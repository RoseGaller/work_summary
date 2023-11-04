# OpenDB

badger/db.go:164

```go
func Open(opt Options) (db *DB, err error) {
   opt.maxBatchSize = (15 * opt.MaxTableSize) / 100
   opt.maxBatchCount = opt.maxBatchSize / int64(skl.MaxNodeSize)

  //LSM dir、valueLog dir
   for _, path := range []string{opt.Dir, opt.ValueDir} {
      dirExists, err := exists(path)
      if err != nil {
         return nil, y.Wrapf(err, "Invalid Dir: %q", path)
      }
      if !dirExists {
         return nil, ErrInvalidDir
      }
   }
  //LSM dir
   absDir, err := filepath.Abs(opt.Dir)
   if err != nil {
      return nil, err
   }
  //ValueLog dir
   absValueDir, err := filepath.Abs(opt.ValueDir)
   if err != nil {
      return nil, err
   }

   dirLockGuard, err := acquireDirectoryLock(opt.Dir, lockFile)
   if err != nil {
      return nil, err
   }
   defer func() {
      if dirLockGuard != nil {
         _ = dirLockGuard.release()
      }
   }()
   var valueDirLockGuard *directoryLockGuard
   if absValueDir != absDir {
      valueDirLockGuard, err = acquireDirectoryLock(opt.ValueDir, lockFile)
      if err != nil {
         return nil, err
      }
   }
   defer func() {
      if valueDirLockGuard != nil {
         _ = valueDirLockGuard.release()
      }
   }()
   if !(opt.ValueLogFileSize <= 2<<30 && opt.ValueLogFileSize >= 1<<20) {
      return nil, ErrValueLogSize
   }
  //打开或者创建MANIFEST文件
   manifestFile, manifest, err := openOrCreateManifestFile(opt.Dir)
   if err != nil {
      return nil, err
   }
   defer func() {
      if manifestFile != nil {
         _ = manifestFile.close()
      }
   }()
	//负责事务
   orc := &oracle{
      isManaged:      opt.managedTxns,
      nextCommit:     1,
      pendingCommits: make(map[uint64]struct{}),
      commits:        make(map[uint64]uint64),
   }
   heap.Init(&orc.commitMark)
	//创建DB
   db = &DB{
      imm:           make([]*skl.Skiplist, 0, opt.NumMemtables),
      flushChan:     make(chan flushTask, opt.NumMemtables),
      writeCh:       make(chan *request, kvWriteChCapacity),
      opt:           opt,
      manifest:      manifestFile,
      elog:          trace.NewEventLog("Badger", "DB"),
      dirLockGuard:  dirLockGuard,
      valueDirGuard: valueDirLockGuard,
      orc:           orc,
   }

   db.closers.updateSize = y.NewCloser(1)
   go db.updateSize(db.closers.updateSize)
  //创建跳表
   db.mt = skl.NewSkiplist(arenaSize(opt))

   // 根据MANIFEST创建levelController
   if db.lc, err = newLevelsController(db, &manifest); err != nil {
      return nil, err
   }
	//开启数据压缩
   db.closers.compactors = y.NewCloser(1)
   db.lc.startCompact(db.closers.compactors)

  //刷新内存中跳表的数据到Level0
   db.closers.memtable = y.NewCloser(1)
   go db.flushMemtable(db.closers.memtable) // Need levels controller to be up.

  //打开VLog
   if err = db.vlog.Open(db, opt); err != nil {
      return nil, err
   }

   headKey := y.KeyWithTs(head, math.MaxUint64)
   // Need to pass with timestamp, lsm get removes the last 8 bytes and compares key
   vs, err := db.get(headKey)
   if err != nil {
      return nil, errors.Wrap(err, "Retrieving head")
   }
   db.orc.curRead = vs.Version
   var vptr valuePointer
   if len(vs.Value) > 0 {
      vptr.Decode(vs.Value)
   }

   replayCloser := y.NewCloser(1)
   go db.doWrites(replayCloser)

   //重放
   if err = db.vlog.Replay(vptr, replayFunction(db)); err != nil {
      return db, err
   }

   replayCloser.SignalAndWait() // Wait for replay to be applied first.
   // Now that we have the curRead, we can update the nextCommit.
   db.orc.nextCommit = db.orc.curRead + 1

   // Mmap writable log
   lf := db.vlog.filesMap[db.vlog.maxFid]
   if err = lf.mmap(2 * db.vlog.opt.ValueLogFileSize); err != nil {
      return db, errors.Wrapf(err, "Unable to mmap RDWR log file")
   }
	//存放写入请求
   db.writeCh = make(chan *request, kvWriteChCapacity)
  //
   db.closers.writes = y.NewCloser(1)
   go db.doWrites(db.closers.writes)

   db.closers.valueGC = y.NewCloser(1)
   go db.vlog.waitOnGC(db.closers.valueGC)

   valueDirLockGuard = nil
   dirLockGuard = nil
   manifestFile = nil
   return db, nil
}
```

# batchSet

badger/db.go:651

```go
func (db *DB) batchSet(entries []*entry) error { //同步写
   //封装写请求，放入writeCh，后台协程将数据写入valuelog和内存中
   req, err := db.sendToWriteCh(entries)  
   if err != nil {
      return err
   }

   req.Wg.Wait()//阻塞
   req.Entries = nil
   err = req.Err
   requestPool.Put(req) //归还对象，放入对象池
   return err
}
```



```go
func (db *DB) sendToWriteCh(entries []*entry) (*request, error) {
   var count, size int64
   for _, e := range entries {
      size += int64(db.opt.estimateSize(e)) //计算单个写入占用的空间
      count++
   }
   
   if count >= db.opt.maxBatchCount || size >= db.opt.maxBatchSize {
      return nil, ErrTxnTooBig
   }
	//从对象池中获取request
   req := requestPool.Get().(*request)
   req.Entries = entries
   req.Wg = sync.WaitGroup{}
   req.Wg.Add(1)
   db.writeCh <- req //放入writeCh
   y.NumPuts.Add(int64(len(entries)))

   return req, nil
}
```

# doWrites

badger/db.go:588

```go
func (db *DB) doWrites(lc *y.Closer) {//将写请求写入ValueLog和内存中
   defer lc.Done()
   pendingCh := make(chan struct{}, 1)

   writeRequests := func(reqs []*request) {
      if err := db.writeRequests(reqs); err != nil {
         log.Printf("ERROR in Badger::writeRequests: %v", err)
      }
      <-pendingCh
   }

   // This variable tracks the number of pending writes.
   reqLen := new(expvar.Int)
   y.PendingWrites.Set(db.opt.Dir, reqLen)

   reqs := make([]*request, 0, 10)
   for {
      var r *request
      select {
      case r = <-db.writeCh: //接收写请求
      case <-lc.HasBeenClosed():
         goto closedCase
      }

      for {
         reqs = append(reqs, r)
         reqLen.Set(int64(len(reqs)))

         if len(reqs) >= 3*kvWriteChCapacity {
            pendingCh <- struct{}{} // blocking.
            goto writeCase
         }

         select {
         // Either push to pending, or continue to pick from writeCh.
         case r = <-db.writeCh:
         case pendingCh <- struct{}{}:
            goto writeCase
         case <-lc.HasBeenClosed():
            goto closedCase
         }
      }

   closedCase:
      close(db.writeCh)
      for r := range db.writeCh { // Flush the channel.
         reqs = append(reqs, r)
      }

      pendingCh <- struct{}{} // Push to pending before doing a write.
      writeRequests(reqs)
      return

   writeCase:
      go writeRequests(reqs)
      reqs = make([]*request, 0, 10)
      reqLen.Set(0)
   }
}
```

```go
 func (db *DB) writeRequests(reqs []*request) error {
   if len(reqs) == 0 {
      return nil
   }

   done := func(err error) {
      for _, r := range reqs {
         r.Err = err
         r.Wg.Done() //唤醒同步写请求
      }
   }

   db.elog.Printf("writeRequests called. Writing to value log")
   //写valueLog
   err := db.vlog.write(reqs) 
   if err != nil {
      done(err)
      return err
   }

   //写内存
   db.elog.Printf("Writing to memtable")
   var count int
   for _, b := range reqs {
      if len(b.Entries) == 0 {
         continue
      }
      count += len(b.Entries)
     //先判断跳表是否有足够空间可写入
      for err := db.ensureRoomForWrite(); err != nil; err = db.ensureRoomForWrite() {
         db.elog.Printf("Making room for writes")
         time.Sleep(10 * time.Millisecond) //休眠
      }
      if err != nil { //空间不足
         done(err)
         return errors.Wrap(err, "writeRequests")
      }
     //写入内存
      if err := db.writeToLSM(b); err != nil {
         done(err)
         return errors.Wrap(err, "writeRequests")
      }
      db.updateOffset(b.Ptrs)
   }
   done(nil) //唤醒同步写请求
   db.elog.Printf("%d entries written", count)
   return nil
}
```

## ensureRoomForWrite

```go
func (db *DB) ensureRoomForWrite() error { //当前写入的跳表是否有足够的空间可写入
   var err error
   db.Lock()
   defer db.Unlock() 
   if db.mt.MemSize() < db.opt.MaxTableSize { 
      return nil //空间足够
   }

   y.AssertTrue(db.mt != nil) // A nil mt indicates that DB is being closed.
   select {
   case db.flushChan <- flushTask{db.mt, db.vptr}: //将内存数据写入Level0
      db.elog.Printf("Flushing value log to disk if async mode.")
      // Ensure value log is synced to disk so this memtable's contents wouldn't be lost.
      err = db.vlog.sync() //valueLog刷盘，确保内存中保存的数据不会丢失
      if err != nil {
         return err
      }

      db.elog.Printf("Flushing memtable, mt.size=%d size of flushChan: %d\n",
         db.mt.MemSize(), len(db.flushChan))
      db.imm = append(db.imm, db.mt) //放入不可写的列表
     //创建新的跳表
      db.mt = skl.NewSkiplist(arenaSize(db.opt))
      return nil
   default:
      return errNoRoom
   }
}
```

## writeValueLog

badger/value.go:672

```go
func (vlog *valueLog) write(reqs []*request) error {
   vlog.filesLock.RLock()
  //当前可写的valueLog
   curlf := vlog.filesMap[vlog.maxFid]
   vlog.filesLock.RUnlock()

   toDisk := func() error {
      if vlog.buf.Len() == 0 { //无数据可写
         return nil
      }
      vlog.elog.Printf("Flushing %d blocks of total size: %d", len(reqs), vlog.buf.Len())
     //写数据
      n, err := curlf.fd.Write(vlog.buf.Bytes())
      if err != nil {
         return errors.Wrapf(err, "Unable to write to value log file: %q", curlf.path)
      }
      y.NumWrites.Add(1)
      y.NumBytesWritten.Add(int64(n))
      vlog.elog.Printf("Done")
      atomic.AddUint32(&vlog.writableLogOffset, uint32(n))
     //清空buf
      vlog.buf.Reset()

     //生成新的valuelog
      if vlog.writableOffset() > uint32(vlog.opt.ValueLogFileSize) {
         var err error
         if err = curlf.doneWriting(vlog.writableLogOffset); err != nil {
            return err
         }
				 //valueLog的标志
         newid := atomic.AddUint32(&vlog.maxFid, 1)
         y.AssertTruef(newid < 1<<16, "newid will overflow uint16: %v", newid)
        //创建VLogFile
         newlf, err := vlog.createVlogFile(newid)
         if err != nil {
            return err
         }

         if err = newlf.mmap(2 * vlog.opt.ValueLogFileSize); err != nil {
            return err
         }

         curlf = newlf
      }
      return nil
   }

   for i := range reqs {
      b := reqs[i]
      b.Ptrs = b.Ptrs[:0]
      for j := range b.Entries { //写入的数据，每个entry都有对应的Ptr
         e := b.Entries[j]
         var p valuePointer

         p.Fid = curlf.fid //valueLog标志
         p.Offset = vlog.writableOffset() + uint32(vlog.buf.Len()) //写入的文件
         plen, err := encodeEntry(e, &vlog.buf) //写入的长度
         if err != nil {
            return err
         }
         p.Len = uint32(plen)
         b.Ptrs = append(b.Ptrs, p)

         if p.Offset > uint32(vlog.opt.ValueLogFileSize) { //达到valueLog写入的上限
            if err := toDisk(); err != nil {//执行刷盘操作
               return err
            }
         }
      }
   }
   return toDisk()
}
```

## writeToLSM

```go
func (db *DB) writeToLSM(b *request) error {
   if len(b.Ptrs) != len(b.Entries) {
      return errors.Errorf("Ptrs and Entries don't match: %+v", b)
   }

   for i, entry := range b.Entries {
      if entry.meta&bitFinTxn != 0 {
         continue
      }
      if db.shouldWriteValueToLSM(*entry) { //判断是否将value保存在LSM中
         db.mt.Put(entry.Key,
            y.ValueStruct{
               Value:     entry.Value, //值
               Meta:      entry.meta,
               UserMeta:  entry.UserMeta,
               ExpiresAt: entry.ExpiresAt,
            })
      } else {
         var offsetBuf [vptrSize]byte
         db.mt.Put(entry.Key,
            y.ValueStruct{
               Value:     b.Ptrs[i].Encode(offsetBuf[:]),//指向ValueLog存放的位置
               Meta:      entry.meta | bitValuePointer,
               UserMeta:  entry.UserMeta,
               ExpiresAt: entry.ExpiresAt,
            })
      }
   }
   return nil
}
```

```go
func (db *DB) shouldWriteValueToLSM(e entry) bool {
   return len(e.Value) < db.opt.ValueThreshold
}
```

# flushMemtable

```go
func (db *DB) flushMemtable(lc *y.Closer) error { //将不可写的跳表写入Level0
   defer lc.Done()

   for ft := range db.flushChan { 
      if ft.mt == nil {
         return nil
      }

      if !ft.mt.Empty() {
         db.elog.Printf("Storing offset: %+v\n", ft.vptr)
         offset := make([]byte, vptrSize)
         ft.vptr.Encode(offset)

         headTs := y.KeyWithTs(head, db.orc.commitTs())
         ft.mt.Put(headTs, y.ValueStruct{Value: offset})
      }

      fileID := db.lc.reserveFileID()
      fd, err := y.CreateSyncedFile(table.NewFilename(fileID, db.opt.Dir), true)
      if err != nil {
         return y.Wrap(err)
      }

      // Don't block just to sync the directory entry.
      dirSyncCh := make(chan error)
      go func() { dirSyncCh <- syncDir(db.opt.Dir) }()

      err = writeLevel0Table(ft.mt, fd)
      dirSyncErr := <-dirSyncCh

      if err != nil {
         db.elog.Errorf("ERROR while writing to level 0: %v", err)
         return err
      }
      if dirSyncErr != nil {
         db.elog.Errorf("ERROR while syncing level directory: %v", dirSyncErr)
         return err
      }

      tbl, err := table.OpenTable(fd, db.opt.TableLoadingMode)
      if err != nil {
         db.elog.Printf("ERROR while opening table: %v", err)
         return err
      }
      // We own a ref on tbl.
      err = db.lc.addLevel0Table(tbl) // This will incrRef (if we don't error, sure)
      tbl.DecrRef()                   // Releases our ref.
      if err != nil {
         return err
      }

      // Update s.imm. Need a lock.
      db.Lock()
      y.AssertTrue(ft.mt == db.imm[0]) //For now, single threaded.
      db.imm = db.imm[1:]
      ft.mt.DecrRef() // Return memory.
      db.Unlock()
   }
   return nil
}
```

## writeLevel0Table

```go
func writeLevel0Table(s *skl.Skiplist, f *os.File) error {//内存数据写入文件
   iter := s.NewIterator()
   defer iter.Close()
   b := table.NewTableBuilder()
   defer b.Close()
   for iter.SeekToFirst(); iter.Valid(); iter.Next() {
      if err := b.Add(iter.Key(), iter.Value()); err != nil {
         return err
      }
   }
   _, err := f.Write(b.Finish())
   return err
}
```

badger/table/builder.go

```go
func (b *Builder) Add(key []byte, value y.ValueStruct) error {
   if b.counter >= restartInterval {//默认100，每100个key形成一个block
      b.finishBlock()
      b.restarts = append(b.restarts, uint32(b.buf.Len()))//restarts表示每个block的起始位置
      b.counter = 0
      b.baseKey = []byte{}
      b.baseOffset = uint32(b.buf.Len())
      b.prevOffset = math.MaxUint32 
   }
   b.addHelper(key, value)
   return nil 
}
```

```go
func (b *Builder) addHelper(key []byte, v y.ValueStruct) {
   // keyBuf主要用来构建布隆过过滤器
   if len(key) > 0 {
      var klen [2]byte
      keyNoTs := y.ParseKey(key)
      binary.BigEndian.PutUint16(klen[:], uint16(len(keyNoTs)))
      b.keyBuf.Write(klen[:])
      b.keyBuf.Write(keyNoTs)
      b.keyCount++
   }

   // 前缀压缩
   var diffKey []byte
   if len(b.baseKey) == 0 {
      b.baseKey = append(b.baseKey[:0], key...)
      diffKey = key
   } else {
      diffKey = b.keyDiff(key)
   }

   h := header{
      plen: uint16(len(key) - len(diffKey)),
      klen: uint16(len(diffKey)),
      vlen: uint16(v.EncodedSize()),
      prev: b.prevOffset, // prevOffset is the location of the last key-value added.
   }
   b.prevOffset = uint32(b.buf.Len()) - b.baseOffset // Remember current offset for the next Add call.

   // Layout: header, diffKey, value.
   var hbuf [10]byte
   h.Encode(hbuf[:])
   b.buf.Write(hbuf[:])
   b.buf.Write(diffKey) // We only need to store the key difference.

   v.EncodeTo(b.buf)
   b.counter++ // Increment number of keys added for this current block.
}
```

```go
func (b *Builder) Finish() []byte {
  //根据key构建布隆过滤器
   bf := bbloom.New(float64(b.keyCount), 0.01)
   var klen [2]byte
   key := make([]byte, 1024)
   for {
      if _, err := b.keyBuf.Read(klen[:]); err == io.EOF {
         break
      } else if err != nil {
         y.Check(err)
      }
      kl := int(binary.BigEndian.Uint16(klen[:]))
      if cap(key) < kl {
         key = make([]byte, 2*kl)
      }
      key = key[:kl]
      y.Check2(b.keyBuf.Read(key))
      bf.Add(key)
   }

   b.finishBlock() // This will never start a new block.
  //写入block的位置信息
   index := b.blockIndex()
   b.buf.Write(index)

   // 写入布隆过滤器信息
   bdata := bf.JSONMarshal()
   n, err := b.buf.Write(bdata)
   y.Check(err)
   var buf [4]byte
   binary.BigEndian.PutUint32(buf[:], uint32(n))
   b.buf.Write(buf[:])
   return b.buf.Bytes()
}
```

## addLevel0Table

```go
func (s *levelsController) addLevel0Table(t *table.Table) error {
   // We update the manifest _before_ the table becomes part of a levelHandler, because at that
   // point it could get used in some compaction.  This ensures the manifest file gets updated in
   // the proper order. (That means this update happens before that of some compaction which
   // deletes the table.)
   err := s.kv.manifest.addChanges([]*protos.ManifestChange{
      makeTableCreateChange(t.ID(), 0),
   })
   if err != nil {
      return err
   }

   for !s.levels[0].tryAddLevel0Table(t) {//level0的文件数量达到上限，进入阻塞
      // Stall. Make sure all levels are healthy before we unstall.
      var timeStart time.Time
      {
         s.elog.Printf("STALLED STALLED STALLED STALLED STALLED STALLED STALLED STALLED: %v\n",
            time.Since(lastUnstalled))
         s.cstatus.RLock()
         for i := 0; i < s.kv.opt.MaxLevels; i++ {
            s.elog.Printf("level=%d. Status=%s Size=%d\n",
               i, s.cstatus.levels[i].debug(), s.levels[i].getTotalSize())
         }
         s.cstatus.RUnlock()
         timeStart = time.Now()
      }
      for {
        //当level0、level1在合并时，进入休眠状态
         if !s.isLevel0Compactable() && !s.levels[1].isCompactable(0) {
            break
         }
         time.Sleep(10 * time.Millisecond)
      }
      {
         s.elog.Printf("UNSTALLED UNSTALLED UNSTALLED UNSTALLED UNSTALLED UNSTALLED: %v\n",
            time.Since(timeStart))
         lastUnstalled = time.Now()
      }
   }
   return nil
}
```

### addChanges

badger/manifest.go

```go
func (mf *manifestFile) addChanges(changesParam []*protos.ManifestChange) error {
  //当有文件增删时，调用此方法
  //manifestFile维护了所有的文件、以及各个levle拥有的文件
   changes := protos.ManifestChangeSet{Changes: changesParam}
   buf, err := changes.Marshal()
   if err != nil {
      return err
   }

   // Maybe we could use O_APPEND instead (on certain file systems)
   mf.appendLock.Lock()
  //修改各个level维护的文件
   if err := applyChangeSet(&mf.manifest, &changes); err != nil {
      mf.appendLock.Unlock()
      return err
   }
   // Rewrite manifest if it'd shrink by 1/10 and it's big enough to care
   if mf.manifest.Deletions > mf.deletionsRewriteThreshold &&
      mf.manifest.Deletions > manifestDeletionsRatio*(mf.manifest.Creations-mf.manifest.Deletions) { //重写manifestFile
      if err := mf.rewrite(); err != nil {
         mf.appendLock.Unlock()
         return err
      }
   } else { //追加写入manifestFile
      var lenCrcBuf [8]byte
      binary.BigEndian.PutUint32(lenCrcBuf[0:4], uint32(len(buf)))
      binary.BigEndian.PutUint32(lenCrcBuf[4:8], crc32.Checksum(buf, y.CastagnoliCrcTable))
      buf = append(lenCrcBuf[:], buf...)
      if _, err := mf.fp.Write(buf); err != nil {
         mf.appendLock.Unlock()
         return err
      }
   }

   mf.appendLock.Unlock()
   return mf.fp.Sync()
}
```

### tryAddLevel0Table

badger/level_handler.go

```go
func (s *levelHandler) tryAddLevel0Table(t *table.Table) bool {
  //将Table放到Level0
   y.AssertTrue(s.level == 0)
   // Need lock as we may be deleting the first table during a level 0 compaction.
   s.Lock()
   defer s.Unlock()
  //level0默认存放10个文件，防止读时遍历所有的文件
   if len(s.tables) >= s.db.opt.NumLevelZeroTablesStall {
      return false
   }

   s.tables = append(s.tables, t)
   t.IncrRef() //增加引用
   s.totalSize += t.Size() //level0文件占用的空间大小

   return true
}
```

# get

badger/db.go:441

```go
func (db *DB) get(key []byte) (y.ValueStruct, error) {
   tables, decr := db.getMemTables() // 获取MemTable，包括可写的和不可写的
   defer decr() //将引用减1

   y.NumGets.Add(1)
  //从MemTable获取数据
   for i := 0; i < len(tables); i++ {
      vs := tables[i].Get(key)
      y.NumMemtableGets.Add(1)
      if vs.Meta != 0 || vs.Value != nil {
         return vs, nil
      }
   }
  //从磁盘LSM文件中获取数据
   return db.lc.get(key)
}
```

## 从memTable查询数据

```go
func (db *DB) getMemTables() ([]*skl.Skiplist, func()) {
   db.RLock()
   defer db.RUnlock()
	//所有不可写的 + 当前可写的
   tables := make([]*skl.Skiplist, len(db.imm)+1)

   // Get mutable memtable.
   tables[0] = db.mt
   tables[0].IncrRef() //引用+1

   //倒序放入tables中，数据由新到旧排序
   last := len(db.imm) - 1
   for i := range db.imm {
      tables[i+1] = db.imm[last-i]
      tables[i+1].IncrRef()//引用+1
   }
   return tables, func() { //方法执行完引用-1
      for _, tbl := range tables {
         tbl.DecrRef()
      }
   }
}
```

## 从LSM文件查询数据

badger/levels.go

```go
func (s *levelsController) get(key []byte) (y.ValueStruct, error) {
   var maxVs y.ValueStruct
   for l, h := range s.levels {
      vs, err := h.get(key) // Calls h.RLock() and h.RUnlock().
      if err != nil {
         return y.ValueStruct{}, errors.Wrapf(err, "get key: %q", key)
      }
      if vs.Value == nil && vs.Meta == 0 {
         continue
      }
      if l == 0 {
         return vs, nil
      }
      if maxVs.Version < vs.Version {
         maxVs = vs
      }
   }
   return maxVs, nil
}
```

badger/level_handler.go

```go
func (s *levelHandler) get(key []byte) (y.ValueStruct, error) {
   //获取key可能存在的文件，每个table对应一个文件
   tables, decr := s.getTableForKey(key)
   keyNoTs := y.ParseKey(key)

   for _, th := range tables {
      if th.DoesNotHave(keyNoTs) { //布隆过滤器
         y.NumLSMBloomHits.Add(s.strLevel, 1)
         continue
      }
			//迭代器
      it := th.NewIterator(false)
      defer it.Close()

      y.NumLSMGets.Add(s.strLevel, 1)
     //定位key
      it.Seek(key)
      if !it.Valid() {
         continue
      }
       //判断key是否相等
      if y.SameKey(key, it.Key()) {
         vs := it.Value()
         vs.Version = y.ParseTs(it.Key())
         return vs, decr()
      }
   }
   return y.ValueStruct{}, decr()
}
```

### 获取key所在的Table

```go
func (s *levelHandler) getTableForKey(key []byte) ([]*table.Table, func() error) {
   s.RLock()
   defer s.RUnlock()
	//level0的文件的数据可能有重叠，需要遍历所有文件
   if s.level == 0 {
      // 倒序遍历
      out := make([]*table.Table, 0, len(s.tables))
      for i := len(s.tables) - 1; i >= 0; i-- {
         out = append(out, s.tables[i])
         s.tables[i].IncrRef() //增加引用
      }
      return out, func() error { //减少引用
         for _, t := range out {
            if err := t.DecrRef(); err != nil {
               return err
            }
         }
         return nil
      }
   }
   //其他level，二分查找key所在的文件
   idx := sort.Search(len(s.tables), func(i int) bool {
      return y.CompareKeys(s.tables[i].Biggest(), key) >= 0
   })
   if idx >= len(s.tables) {
      return nil, func() error { return nil }
   }
   tbl := s.tables[idx]
   tbl.IncrRef()//增加引用
   return []*table.Table{tbl}, tbl.DecrRef
}
```

### 定位Key所属的Block

badger/table/iterator.go

```go
func (itr *Iterator) Seek(key []byte) {
   if !itr.reversed { //查找大于等于key
      itr.seek(key)
   } else {
      itr.seekForPrev(key) //查找小于等于key
   }
}
```

```go
func (itr *Iterator) seek(key []byte) { //查找大于等于key，往后查找
   itr.seekFrom(key, origin)
}
```

```go
func (itr *Iterator) seekFrom(key []byte, whence int) {
   itr.err = nil
   switch whence {
   case origin:
      itr.reset()
   case current:
   }
	//二分查找Key所在的Block
   idx := sort.Search(len(itr.t.blockIndex), func(idx int) bool {
      ko := itr.t.blockIndex[idx]
      return y.CompareKeys(ko.key, key) > 0
   })
   if idx == 0 {
      itr.seekHelper(0, key)
      return
   }

  	//查询block[idx-1]、block[idx]
   // block[idx].smallest is > key.
   // Since idx>0, we know block[idx-1].smallest is <= key.
   // There are two cases.
   // 1) Everything in block[idx-1] is strictly < key. In this case, we should go to the first element of block[idx].
   // 2) Some element in block[idx-1] is >= key. We should go to that element.
   itr.seekHelper(idx-1, key)
   if itr.err == io.EOF {
      // Case 1. Need to visit block[idx].
      if idx == len(itr.t.blockIndex) {
        	//查询的Key大于所有已经存在的Key
         return
      }
      itr.seekHelper(idx, key)
   }
}
```



```go
func (itr *Iterator) seekHelper(blockIdx int, key []byte) {
   //定位Key所属的block
   itr.bpos = blockIdx
   block, err := itr.t.block(blockIdx)
   if err != nil {
      itr.err = err
      return
   }
 	//创建block迭代器
   itr.bi = block.NewIterator()
   //定位在block中的位置
   itr.bi.Seek(key, origin)
   itr.err = itr.bi.Error()
}
```

### 定位Key的在Block中的位置

```go
func (itr *blockIterator) Seek(key []byte, whence int) {
   itr.err = nil

   switch whence {
   case origin:
      itr.Reset()
   case current:
   }

   var done bool
   // 遍历block
   for itr.Init(); itr.Valid(); itr.Next() {
      k := itr.Key()
      if y.CompareKeys(k, key) >= 0 {
         // We are done as k is >= key.
         done = true
         break
      }
   }
   if !done {
      itr.err = io.EOF //未查询到此Key
   }
}
```

```go
func (itr *blockIterator) Init() {
   if !itr.init {//调用reset时，将其置为false
      itr.Next()
   }
}
```

```go
func (itr *blockIterator) Next() {
   itr.init = true
   itr.err = nil
   if itr.pos >= uint32(len(itr.data)) { //越界，在block中没有查询到此key
      itr.err = io.EOF
      return
   }

   var h header
   itr.pos += uint32(h.Decode(itr.data[itr.pos:])) //10个字节
   itr.last = h // Store the last header.

   if h.klen == 0 && h.plen == 0 {
      // Last entry in the table.
      itr.err = io.EOF
      return
   }

   // Populate baseKey if it isn't set yet. This would only happen for the first Next.
   if len(itr.baseKey) == 0 {
      // This should be the first Next() for this block. Hence, prefix length should be zero.
      y.AssertTrue(h.plen == 0)
      itr.baseKey = itr.data[itr.pos : itr.pos+uint32(h.klen)]
   }
   itr.parseKV(h)
}
```

# Compact

开启多个协程执行合并操作

badger/levels.go

```go
func (s *levelsController) startCompact(lc *y.Closer) { //在打开DB时，调用此方法
   n := s.kv.opt.NumCompactors //默认开启三个协程处理合并操作
   lc.AddRunning(n - 1)
   for i := 0; i < n; i++ {
      go s.runWorker(lc)
   }
}
```



```go
func (s *levelsController) runWorker(lc *y.Closer) {
   defer lc.Done()
   if s.kv.opt.DoNotCompact {
      return
   }

   time.Sleep(time.Duration(rand.Int31n(1000)) * time.Millisecond)
   ticker := time.NewTicker(time.Second)
   defer ticker.Stop()

   for {
      select {
      case <-ticker.C:
         //挑选合并的level，根据level的优先级进行排序
         prios := s.pickCompactLevels()
         for _, p := range prios {
            didCompact, _ := s.doCompact(p) //执行合并
            if didCompact {
               break
            }
         }
      case <-lc.HasBeenClosed():
         return
      }
   }

```

## pickCompactLevels

```go
func (s *levelsController) pickCompactLevels() (prios []compactionPriority) {

  	//level0根据文件的个数计算优先级	
   if !s.cstatus.overlapsWith(0, infRange) && s.isLevel0Compactable() {
      pri := compactionPriority{
         level: 0,
         score: float64(s.levels[0].numTables()) / float64(s.kv.opt.NumLevelZeroTables),
      }
      prios = append(prios, pri)
   }
   //其他level根据占用的空间计算优先级
   for i, l := range s.levels[1:] {
      delSize := s.cstatus.delSize(i + 1)
      if l.isCompactable(delSize) {
         pri := compactionPriority{
            level: i + 1,
            score: float64(l.getTotalSize()-delSize) / float64(l.maxTotalSize),
         }
         prios = append(prios, pri)
      }
   }
  //根据优先级排序
   sort.Slice(prios, func(i, j int) bool {
      return prios[i].score > prios[j].score
   })
   return prios
}
```

## doCompact

```go
func (s *levelsController) doCompact(p compactionPriority) (bool, error) {
   l := p.level
   y.AssertTrue(l+1 < s.kv.opt.MaxLevels) //最高的level不会和下一级level合并

  //参与合并的level及合并的区间
   cd := compactDef{
      elog:      trace.New("Badger", "Compact"),
      thisLevel: s.levels[l],
      nextLevel: s.levels[l+1],
   }
   cd.elog.SetMaxEvents(100)
   defer cd.elog.Finish()

   cd.elog.LazyPrintf("Got compaction priority: %+v", p)

   // While picking tables to be compacted, both levels' tables are expected to
   // remain unchanged.
  //填充compactDef属性，获取合并的table
  //当挑选table进行合并时，需要保证两个level是保持不变的
   if l == 0 {
      if !s.fillTablesL0(&cd) {
         cd.elog.LazyPrintf("fillTables failed for level: %d\n", l)
         return false, nil
      }

   } else {
      if !s.fillTables(&cd) {
         cd.elog.LazyPrintf("fillTables failed for level: %d\n", l)
         return false, nil
      }
   }

   cd.elog.LazyPrintf("Running for level: %d\n", cd.thisLevel.level)
   s.cstatus.toLog(cd.elog)
   if err := s.runCompactDef(l, cd); err != nil {
      // This compaction couldn't be done successfully.
      cd.elog.LazyPrintf("\tLOG Compact FAILED with error: %+v: %+v", err, cd)
      return false, err
   }

   // Done with compaction. So, remove the ranges from compaction status.
   s.cstatus.delete(cd)
   s.cstatus.toLog(cd.elog)
   cd.elog.LazyPrintf("Compaction for level: %d DONE", cd.thisLevel.level)
   return true, nil
}
```

### fillTablesL0

```go
func (s *levelsController) fillTablesL0(cd *compactDef) bool {
   cd.lockLevels()//对参与合并的thisLevel和nextLevel加读锁
   defer cd.unlockLevels()

  //top存放thisLevel的table
   cd.top = make([]*table.Table, len(cd.thisLevel.tables))
   copy(cd.top, cd.thisLevel.tables)
   if len(cd.top) == 0 {
      return false
   }
   cd.thisRange = infRange
	//获取thisLevel的key的区间范围
   kr := getKeyRange(cd.top)
  //获取nextLevel中有数据重叠的Tables
   left, right := cd.nextLevel.overlappingTables(levelHandlerRLocked{}, kr)
   cd.bot = make([]*table.Table, right-left)
   copy(cd.bot, cd.nextLevel.tables[left:right])

   if len(cd.bot) == 0 {//无数据重叠
      cd.nextRange = kr
   } else {
      cd.nextRange = getKeyRange(cd.bot) //重叠table的key的范围
   }
	//是否可以合并
   if !s.cstatus.compareAndAdd(thisAndNextLevelRLocked{}, *cd) {
      return false
   }

   return true
}
```

badger/compaction.go

```go
func (cs *compactStatus) compareAndAdd(_ thisAndNextLevelRLocked, cd compactDef) bool {
   cs.Lock() //加锁
   defer cs.Unlock()

   level := cd.thisLevel.level

   y.AssertTruef(level < len(cs.levels)-1, "Got level %d. Max levels: %d", level, len(cs.levels))
   thisLevel := cs.levels[level]
   nextLevel := cs.levels[level+1]

   if thisLevel.overlapsWith(cd.thisRange) { //thisLevel此区间是否正在合并
      return false
   }
   if nextLevel.overlapsWith(cd.nextRange) { //nextLevel此区间是否正在合并
      return false
   }
   // 检测thisLevel是否需要合并
   if cd.thisLevel.totalSize-thisLevel.delSize < cd.thisLevel.maxTotalSize {
      return false
   }

  //设置thisLevel、nextLevel正在合并的区间
   thisLevel.ranges = append(thisLevel.ranges, cd.thisRange)
   nextLevel.ranges = append(nextLevel.ranges, cd.nextRange)
  //thisLevel需要删除的空间大小
   thisLevel.delSize += cd.thisSize

   return true 
}
```

### fillTables

badger/levels.go

```go
func (s *levelsController) fillTables(cd *compactDef) bool {
   cd.lockLevels()
   defer cd.unlockLevels()

   tbls := make([]*table.Table, len(cd.thisLevel.tables))
   copy(tbls, cd.thisLevel.tables)
   if len(tbls) == 0 {
      return false
   }

   // 优先合并大文件
   sort.Slice(tbls, func(i, j int) bool {
      return tbls[i].Size() > tbls[j].Size()
   })

   for _, t := range tbls {
      cd.thisSize = t.Size()
      cd.thisRange = keyRange{
         left:  t.Smallest(),
         right: t.Biggest(),
      }
      if s.cstatus.overlapsWith(cd.thisLevel.level, cd.thisRange) {
         continue
      }
      //thisLevel只合并一个文件
      cd.top = []*table.Table{t}
     //nextLevel重叠的文件
      left, right := cd.nextLevel.overlappingTables(levelHandlerRLocked{}, cd.thisRange)
      cd.bot = make([]*table.Table, right-left)
      copy(cd.bot, cd.nextLevel.tables[left:right])

      if len(cd.bot) == 0 { //无重叠
         cd.bot = []*table.Table{}
         cd.nextRange = cd.thisRange
         if !s.cstatus.compareAndAdd(thisAndNextLevelRLocked{}, *cd) {
            continue
         }
         return true
      }
       //重叠件的区间范围
      cd.nextRange = getKeyRange(cd.bot)

     //nextLevl的nextRange是否正在参与合并
      if s.cstatus.overlapsWith(cd.nextLevel.level, cd.nextRange) {
         continue
      }

      if !s.cstatus.compareAndAdd(thisAndNextLevelRLocked{}, *cd) {
         continue
      }
      return true
   }
   return false
}
```

```go
func (s *levelsController) runCompactDef(l int, cd compactDef) (err error) {
   timeStart := time.Now()

   thisLevel := cd.thisLevel
   nextLevel := cd.nextLevel
	
   if thisLevel.level >= 1 && len(cd.bot) == 0 {
     //非level-0合并且nextLevel无重叠文件
      y.AssertTrue(len(cd.top) == 1)
      tbl := cd.top[0] //非level-0合并，thisLevel只选择一个Table进行合并
			//修改MAINFEST：thisLevel删除文件、nextLevel增加文件
      changes := []*protos.ManifestChange{
         makeTableDeleteChange(tbl.ID()),
         makeTableCreateChange(tbl.ID(), nextLevel.level),
      }
      if err := s.kv.manifest.addChanges(changes); err != nil {
         return err
      }

			//先替换后删除，避免读时出现问题
      if err := nextLevel.replaceTables(cd.top); err != nil {
         return err
      }
      if err := thisLevel.deleteTables(cd.top); err != nil {
         return err
      }

      cd.elog.LazyPrintf("\tLOG Compact-Move %d->%d smallest:%s biggest:%s took %v\n",
         l, l+1, string(tbl.Smallest()), string(tbl.Biggest()), time.Since(timeStart))
      return nil
   }

   newTables, decr, err := s.compactBuildTables(l, cd)
   if err != nil {
      return err
   }
   defer func() {
      // Only assign to err, if it's not already nil.
      if decErr := decr(); err == nil {
         err = decErr
      }
   }()
   changeSet := buildChangeSet(&cd, newTables)

   // We write to the manifest _before_ we delete files (and after we created files)
   if err := s.kv.manifest.addChanges(changeSet.Changes); err != nil {
      return err
   }

   // See comment earlier in this function about the ordering of these ops, and the order in which
   // we access levels when reading.
   if err := nextLevel.replaceTables(newTables); err != nil {
      return err
   }
   if err := thisLevel.deleteTables(cd.top); err != nil {
      return err
   }

   // Note: For level 0, while doCompact is running, it is possible that new tables are added.
   // However, the tables are added only to the end, so it is ok to just delete the first table.

   cd.elog.LazyPrintf("LOG Compact %d->%d, del %d tables, add %d tables, took %v\n",
      l, l+1, len(cd.top)+len(cd.bot), len(newTables), time.Since(timeStart))
   return nil
}
```

```go
func (s *levelHandler) replaceTables(newTables []*table.Table) error {
   // Need to re-search the range of tables in this level to be replaced as other goroutines might
   // be changing it as well.  (They can't touch our tables, but if they add/remove other tables,
   // the indices get shifted around.)
   if len(newTables) == 0 {
      return nil
   }

   s.Lock() // We s.Unlock() below.

   // Increase totalSize first.
   for _, tbl := range newTables {
      s.totalSize += tbl.Size()
      tbl.IncrRef()
   }

   kr := keyRange{
      left:  newTables[0].Smallest(),
      right: newTables[len(newTables)-1].Biggest(),
   }
   left, right := s.overlappingTables(levelHandlerRLocked{}, kr)

   toDecr := make([]*table.Table, right-left)
   // Update totalSize and reference counts.
   for i := left; i < right; i++ {
      tbl := s.tables[i]
      s.totalSize -= tbl.Size()
      toDecr[i-left] = tbl
   }

   // To be safe, just make a copy. TODO: Be more careful and avoid copying.
   numDeleted := right - left
   numAdded := len(newTables)
   tables := make([]*table.Table, len(s.tables)-numDeleted+numAdded)
   y.AssertTrue(left == copy(tables, s.tables[:left]))
   t := tables[left:]
   y.AssertTrue(numAdded == copy(t, newTables))
   t = t[numAdded:]
   y.AssertTrue(len(s.tables[right:]) == copy(t, s.tables[right:]))
   s.tables = tables
   s.Unlock() // s.Unlock before we DecrRef tables -- that can be slow.
   return decrRefs(toDecr)
}
```

badger/levels.go

```go
func (s *levelsController) compactBuildTables(
   l int, cd compactDef) ([]*table.Table, func() error, error) {
   topTables := cd.top
   botTables := cd.bot

   //创建迭代器
   var iters []y.Iterator
   if l == 0 {
      //level-0倒序排列参与合并的table
      iters = appendIteratorsReversed(iters, topTables, false)
   } else {
      //其他level，只有一个table参与合并
      y.AssertTrue(len(topTables) == 1)
      iters = []y.Iterator{topTables[0].NewIterator(false)}
   }

   // Next level has level>=1 and we can use ConcatIterator as key ranges do not overlap.
   iters = append(iters, table.NewConcatIterator(botTables, false))
   it := y.NewMergeIterator(iters, false)
   defer it.Close() // Important to close the iterator to do ref counting.

   it.Rewind()

   // Start generating new tables.
   type newTableResult struct {
      table *table.Table
      err   error
   }
   resultCh := make(chan newTableResult)
   var i int
   for ; it.Valid(); i++ {
      timeStart := time.Now()
      builder := table.NewTableBuilder()
      for ; it.Valid(); it.Next() {
         if builder.ReachedCapacity(s.kv.opt.MaxTableSize) {
            break
         }
         y.Check(builder.Add(it.Key(), it.Value()))
      }
      // It was true that it.Valid() at least once in the loop above, which means we
      // called Add() at least once, and builder is not Empty().
      y.AssertTrue(!builder.Empty())

      cd.elog.LazyPrintf("LOG Compact. Iteration to generate one table took: %v\n", time.Since(timeStart))
			//生成文件Id
      fileID := s.reserveFileID()
      go func(builder *table.Builder) {
         defer builder.Close()
					//创建文件
         fd, err := y.CreateSyncedFile(table.NewFilename(fileID, s.kv.opt.Dir), true)
         if err != nil {
            resultCh <- newTableResult{nil, errors.Wrapf(err, "While opening new table: %d", fileID)}
            return
         }	
				//写文件
         if _, err := fd.Write(builder.Finish()); err != nil {
            resultCh <- newTableResult{nil, errors.Wrapf(err, "Unable to write to file: %d", fileID)}
            return
         }
			
         tbl, err := table.OpenTable(fd, s.kv.opt.TableLoadingMode)
         // decrRef is added below.
         resultCh <- newTableResult{tbl, errors.Wrapf(err, "Unable to open table: %q", fd.Name())}
      }(builder)
   }

   newTables := make([]*table.Table, 0, 20)

   // Wait for all table builders to finish.
   var firstErr error
   for x := 0; x < i; x++ {
      res := <-resultCh
      newTables = append(newTables, res.table)
      if firstErr == nil {
         firstErr = res.err
      }
   }

   if firstErr == nil {
      // Ensure created files' directory entries are visible.  We don't mind the extra latency
      // from not doing this ASAP after all file creation has finished because this is a
      // background operation.
      firstErr = syncDir(s.kv.opt.Dir)
   }

   if firstErr != nil {
      // An error happened.  Delete all the newly created table files (by calling DecrRef
      // -- we're the only holders of a ref).
      for j := 0; j < i; j++ {
         if newTables[j] != nil {
            newTables[j].DecrRef()
         }
      }
      errorReturn := errors.Wrapf(firstErr, "While running compaction for: %+v", cd)
      return nil, nil, errorReturn
   }

   sort.Slice(newTables, func(i, j int) bool {
      return y.CompareKeys(newTables[i].Biggest(), newTables[j].Biggest()) < 0
   })

   return newTables, func() error { return decrRefs(newTables) }, nil
}
```

# TXN

## NewTransaction

badger/transaction.go:435

```go
func (db *DB) NewTransaction(update bool) *Txn {
   txn := &Txn{
     update: update, //true：可写 ,false:只读
      db:     db,
      readTs: db.orc.readTs(),
      count:  1,                       // One extra entry for BitFin.
      size:   int64(len(txnKey) + 10), // Some buffer for the extra entry.
   }
   if update {
      txn.pendingWrites = make(map[string]*entry) //待写数据
      txn.db.orc.addRef() //orc引用+1
   }
   return txn
}
```

## Set

badger/transaction.go:206

```go
func (txn *Txn) Set(key, val []byte) error {
   e := &entry{
      Key:   key,
      Value: val,
   }
   return txn.setEntry(e)
}
```

```go
func (txn *Txn) setEntry(e *entry) error {
   switch {
   case !txn.update:
      return ErrReadOnlyTxn
   case txn.discarded:
      return ErrDiscardedTxn
   case len(e.Key) == 0:
      return ErrEmptyKey
   case len(e.Key) > maxKeySize:
      return exceedsMaxKeySizeError(e.Key)
   case int64(len(e.Value)) > txn.db.opt.ValueLogFileSize:
      return exceedsMaxValueSizeError(e.Value, txn.db.opt.ValueLogFileSize)
   }
   if err := txn.checkSize(e); err != nil {
      return err
   }

   fp := farm.Fingerprint64(e.Key) 
   txn.writes = append(txn.writes, fp) //当前事务写入的key
   txn.pendingWrites[string(e.Key)] = e //当前事务写入的数据
   return nil
}
```

badger/transaction.go:371

```go
func (txn *Txn) Commit(callback func(error)) error {
   if txn.commitTs == 0 && txn.db.opt.managedTxns {
      return ErrManagedTxn
   }
   if txn.discarded { 
      return ErrDiscardedTxn
   }
   defer txn.Discard()
   if len(txn.writes) == 0 { //无数据可写
      return nil
   }

   state := txn.db.orc
  //提交时间戳
   commitTs := state.newCommitTs(txn)
   if commitTs == 0 {
      return ErrConflict
   }
   defer state.doneCommit(commitTs)

   entries := make([]*entry, 0, len(txn.pendingWrites)+1)
   for _, e := range txn.pendingWrites {
     // key = key + (math.MaxUint64 - commitTS)
      e.Key = y.KeyWithTs(e.Key, commitTs)
      e.meta |= bitTxn
      entries = append(entries, e)
   }
  //表明事务结束
   e := &entry{
      Key:   y.KeyWithTs(txnKey, commitTs),
      Value: []byte(strconv.FormatUint(commitTs, 10)),
      meta:  bitFinTxn,
   }
   entries = append(entries, e)

   if callback == nil {
      return txn.db.batchSet(entries) //同步
   }
   return txn.db.batchSetAsync(entries, callback) //异步
```

## Get

```go
func (txn *Txn) Get(key []byte) (item *Item, rerr error) {
   if len(key) == 0 {
      return nil, ErrEmptyKey
   } else if txn.discarded {
      return nil, ErrDiscardedTxn
   }

   item = new(Item)
   if txn.update { //当前事务可写
      if e, has := txn.pendingWrites[string(key)]; has && bytes.Equal(key, e.Key) {
        //当前事务写入此key
         item.meta = e.meta
         item.val = e.Value
         item.userMeta = e.UserMeta
         item.key = key
         item.status = prefetched
         item.version = txn.readTs
         return item, nil
      }
     
      fp := farm.Fingerprint64(key)
      txn.reads = append(txn.reads, fp) //追踪此事务读的key
   }
  //key = key + (math.MaxUint64 - readTs)
   seek := y.KeyWithTs(key, txn.readTs)
   vs, err := txn.db.get(seek)
   if err != nil {
      return nil, errors.Wrapf(err, "DB::Get key: %q", key)
   }
   if vs.Value == nil && vs.Meta == 0 {
      return nil, ErrKeyNotFound
   }
   if isDeletedOrExpired(vs) {
      return nil, ErrKeyNotFound
   }

   item.key = key
   item.version = vs.Version
   item.meta = vs.Meta
   item.userMeta = vs.UserMeta
   item.db = txn.db
   item.vptr = vs.Value
   item.txn = txn
   return item, nil
}
```

# References

https://www.usenix.org/system/files/conference/fast16/fast16-papers-lu.pdf