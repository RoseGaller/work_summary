# 概述

Bitcask存储模型，写时顺序写，然后建立索引信息。

索引放在内存中，通过Adaptive Radix Tree进行存储,并且支持磁盘持久化

# Open

bitcask.go

```go
// Option is a function that takes a config struct and modifies it
type Option func(*config.Config) error
```

```go
func Open(path string, options ...Option) (*Bitcask, error) {
   var (
      cfg  *config.Config
      err  error
      meta *metadata.MetaData
   )

   configPath := filepath.Join(path, "config.json")
   if internal.Exists(configPath) { //路径存在，加载配置信息
      cfg, err = config.Load(configPath)
      if err != nil {
         return nil, err
      }
   } else {
      cfg = newDefaultConfig() //创建默认的配置
   }
  //检测是否升级
   if err := checkAndUpgrade(cfg, configPath); err != nil {
      return nil, err
   }
  //修改配置信息
   for _, opt := range options {
      if err := opt(cfg); err != nil {
         return nil, err
      }
   }
   if err := os.MkdirAll(path, cfg.DirFileModeBeforeUmask); err != nil {
      return nil, err
   }
	//加载metadata
   meta, err = loadMetadata(path)
   if err != nil {
      return nil, err
   }
	//创建Bitcask
   bitcask := &Bitcask{
      Flock:    flock.New(filepath.Join(path, lockfile)),
      config:   cfg,
      options:  options,
      path:     path,
      indexer:  index.NewIndexer(),
      metadata: meta,
   }

   locked, err := bitcask.Flock.TryLock()
   if err != nil {
      return nil, err
   }

   if !locked {
      return nil, ErrDatabaseLocked
   }
	//保存配置
   if err := cfg.Save(configPath); err != nil {
      return nil, err
   }
	//自动恢复
   if cfg.AutoRecovery {
      if err := data.CheckAndRecover(path, cfg); err != nil {
         return nil, fmt.Errorf("recovering database: %s", err)
      }
   }
   if err := bitcask.Reopen(); err != nil {
      return nil, err
   }

   return bitcask, nil
}
```



```go
func CheckAndRecover(path string, cfg *config.Config) error {
   dfs, err := internal.GetDatafiles(path)
   if err != nil {
      return fmt.Errorf("scanning datafiles: %s", err)
   }
   if len(dfs) == 0 {
      return nil
   }
   f := dfs[len(dfs)-1]
   recovered, err := recoverDatafile(f, cfg)
   if err != nil {
      return fmt.Errorf("recovering data file")
   }
   if recovered {
      if err := os.Remove(filepath.Join(path, "index")); err != nil {
         return fmt.Errorf("error deleting the index on recovery: %s", err)
      }
   }
   return nil
}
```

## reopen

bitcask.go

```go
func (b *Bitcask) Reopen() error {
   b.mu.Lock()
   defer b.mu.Unlock()

   return b.reopen()
}
```



```go
func (b *Bitcask) reopen() error {
  //加载数据文件
   datafiles, lastID, err := loadDatafiles(b.path, b.config.MaxKeySize, b.config.MaxValueSize, b.config.FileFileModeBeforeUmask)
   if err != nil {
      return err
   }
  //加载索引文件
   t, err := loadIndex(b.path, b.indexer, b.config.MaxKeySize, datafiles, lastID, b.metadata.IndexUpToDate)
   if err != nil {
      return err
   }
	//非只读；创建当前写入的文件，即上次写入的文件
   curr, err := data.NewDatafile(b.path, lastID, false, b.config.MaxKeySize, b.config.MaxValueSize, b.config.FileFileModeBeforeUmask)
   if err != nil {
      return err
   }

   b.trie = t//索引树
   b.curr = curr //当前可写入的文件
   b.datafiles = datafiles //维护所有的数据文件（包括正在写入的文件）

   return nil
}
```

### loadDatafiles

```go
func loadDatafiles(path string, maxKeySize uint32, maxValueSize uint64, fileModeBeforeUmask os.FileMode) (datafiles map[int]data.Datafile, lastID int, err error) {
  	//加载目录下的所有的数据文件，并排序
   fns, err := internal.GetDatafiles(path)
   if err != nil {
      return nil, 0, err
   }
	//解析文件名中的Id
   ids, err := internal.ParseIds(fns)
   if err != nil {
      return nil, 0, err
   }

  //创建datafile
   datafiles = make(map[int]data.Datafile, len(ids))
   for _, id := range ids {
     //只读
      datafiles[id], err = data.NewDatafile(path, id, true, maxKeySize, maxValueSize, fileModeBeforeUmask)
      if err != nil {
         return
      }

   }
  //最新的文件Id
   if len(ids) > 0 {
      lastID = ids[len(ids)-1]
   }
   return
}
```

internal/data/datafile.go

```go
func NewDatafile(path string, id int, readonly bool, maxKeySize uint32, maxValueSize uint64, fileMode os.FileMode) (Datafile, error) {
   var (
      r   *os.File
      ra  *mmap.ReaderAt
      w   *os.File
      err error
   )

   fn := filepath.Join(path, fmt.Sprintf(defaultDatafileFilename, id))

   if !readonly {  //非只读
      w, err = os.OpenFile(fn, os.O_WRONLY|os.O_APPEND|os.O_CREATE, fileMode)
      if err != nil {
         return nil, err
      }
   }

   r, err = os.Open(fn) //只读
   if err != nil {
      return nil, err
   }
   stat, err := r.Stat()
   if err != nil {
      return nil, errors.Wrap(err, "error calling Stat()")
   }

   ra, err = mmap.Open(fn)
   if err != nil {
      return nil, err
   }

   offset := stat.Size()

   dec := codec.NewDecoder(r, maxKeySize, maxValueSize) //读取
   enc := codec.NewEncoder(w) //写入

   return &datafile{
      id:           id,
      r:            r,
      ra:           ra,
      w:            w,
      offset:       offset,
      dec:          dec,
      enc:          enc,
      maxKeySize:   maxKeySize,
      maxValueSize: maxValueSize,
   }, nil
}
```

### loadIndex

```go
func loadIndex(path string, indexer index.Indexer, maxKeySize uint32, datafiles map[int]data.Datafile, lastID int, indexUpToDate bool) (art.Tree, error) {
  //加载索引信息
   t, found, err := indexer.Load(filepath.Join(path, "index"), maxKeySize)
   if err != nil {
      return nil, err
   }
   if found && indexUpToDate { //写入数据都已建立索引
      return t, nil
   }
  
   if found {//当前写入的文件创建索引
      if err := loadIndexFromDatafile(t, datafiles[lastID]); err != nil {
         return nil, err
      }
      return t, nil
   }
  //按数据文件名称排序，最新写入的数据要覆盖旧的数据
   sortedDatafiles := getSortedDatafiles(datafiles)
   //所有的文件建立索引
   for _, df := range sortedDatafiles {
      if err := loadIndexFromDatafile(t, df); err != nil {
         return nil, err
      }
   }
   return t, nil
}
```

Internal/index/index.go

```go
func (i *indexer) Load(path string, maxKeySize uint32) (art.Tree, bool, error) {
   t := art.New()

   if !internal.Exists(path) {
      return t, false, nil //索引文件不存在
   }

   f, err := os.Open(path) //只读模式打开
   if err != nil {
      return t, true, err
   }
   defer f.Close()

   if err := readIndex(f, t, maxKeySize); err != nil {//读取索引文件
      return t, true, err
   }
   return t, true, nil
}
```

intenal/index/codec_index.go

```go
func readIndex(r io.Reader, t art.Tree, maxKeySize uint32) error {
   for {
      key, err := readKeyBytes(r, maxKeySize)//读取key
      if err != nil {
         if err == io.EOF {
            break
         }
         return err
      }

      item, err := readItem(r) //读取key所在的文件信息
      if err != nil {
         return err
      }

      t.Insert(key, item)
   }

   return nil
}
```

```go
func readKeyBytes(r io.Reader, maxKeySize uint32) ([]byte, error) {
  //读取key的字节数
   s := make([]byte, int32Size)
   _, err := io.ReadFull(r, s)
   if err != nil {
      if err == io.EOF {
         return nil, err
      }
      return nil, errors.Wrap(errTruncatedKeySize, err.Error())
   }
   size := binary.BigEndian.Uint32(s)
   if maxKeySize > 0 && size > uint32(maxKeySize) {
      return nil, errKeySizeTooLarge
   }

  //读取key的内容
   b := make([]byte, size)
   _, err = io.ReadFull(r, b)
   if err != nil {
      return nil, errors.Wrap(errTruncatedKeyData, err.Error())
   }
   return b, nil
}
```

```go
func readItem(r io.Reader) (internal.Item, error) {
  //4+8+8
   buf := make([]byte, (fileIDSize + offsetSize + sizeSize))
   _, err := io.ReadFull(r, buf)
   if err != nil {
      return internal.Item{}, errors.Wrap(errTruncatedData, err.Error())
   }

   return internal.Item{
      FileID: int(binary.BigEndian.Uint32(buf[:fileIDSize])),
      Offset: int64(binary.BigEndian.Uint64(buf[fileIDSize:(fileIDSize + offsetSize)])),
      Size:   int64(binary.BigEndian.Uint64(buf[(fileIDSize + offsetSize):])),
   }, nil
}
```



# Put

bitcask.go

```go
func (b *Bitcask) Put(key, value []byte, options ...PutOptions) error {
   //检测key是否合法
   if len(key) == 0 {
      return ErrEmptyKey
   }
   if b.config.MaxKeySize > 0 && uint32(len(key)) > b.config.MaxKeySize {
      return ErrKeyTooLarge
   }
  //检测value受合法
   if b.config.MaxValueSize > 0 && uint64(len(value)) > b.config.MaxValueSize {
      return ErrValueTooLarge
   }
   var feature Feature
   for _, opt := range options {
      if err := opt(&feature); err != nil {
         return err
      }
   }
  //加锁
   b.mu.Lock()
   defer b.mu.Unlock() //方法执行完释放锁
   //写入文件，返回写入文件的位置
     offset, n, err := b.put(key, value, feature)
   if err != nil {
      return err
   }
   //是否同步刷盘
   if b.config.Sync {
      if err := b.curr.Sync(); err != nil {
         return err
      }
   }
   // in case of successful `put`, IndexUpToDate will be always be false
   b.metadata.IndexUpToDate = false
   //索引中查询key是否存在
   if oldItem, found := b.trie.Search(key); found {
      //统计可回收的空间，key之前对应的值已无效
      b.metadata.ReclaimableSpace += oldItem.(internal.Item).Size
   }
   //索引信息:文件Id、文件中的位置offset、值的长度size
   item := internal.Item{FileID: b.curr.FileID(), Offset: offset, Size: n}
   //插入索引信息
   b.trie.Insert(key, item)
   return nil
}
```

bitcask.go:

```go
func (b *Bitcask) put(key, value []byte, feature Feature) (int64, int64, error) {//写文件，返回offset
   size := b.curr.Size()
   //当前文件剩余空间不足
   if size >= int64(b.config.MaxDatafileSize) {
      //关闭当前文件
      err := b.curr.Close()
      if err != nil {
         return -1, 0, err
      }
      //当前文件变成只读状态，datafiles维护 fileId -> DataFile
      id := b.curr.FileID()
      df, err := data.NewDatafile(b.path, id, true, b.config.MaxKeySize, b.config.MaxValueSize, b.config.FileFileModeBeforeUmask)
      if err != nil {
         return -1, 0, err
      }
      b.datafiles[id] = df
      //生成新文件
      id = b.curr.FileID() + 1
      curr, err := data.NewDatafile(b.path, id, false, b.config.MaxKeySize, b.config.MaxValueSize, b.config.FileFileModeBeforeUmask)
      if err != nil {
         return -1, 0, err
      }
      b.curr = curr
      //保存索引到磁盘文件
      err = b.saveIndex()
      if err != nil {
         return -1, 0, err
      }
   }
   //封装成Entry
   e := internal.NewEntry(key, value, feature.Expiry)
   //写文件
   return b.curr.Write(e)
}
```

internal/data/datafile.go

```go
func (df *datafile) Write(e internal.Entry) (int64, int64, error) {
   if df.w == nil {
      return -1, 0, errReadonly
   }
  //加锁
   df.Lock()
   defer df.Unlock()
   e.Offset = df.offset
   n, err := df.enc.Encode(e)
   if err != nil {
      return -1, 0, err
   }
   df.offset += n
   return e.Offset, n, nil //返回写入的文件位置、数据大小
}
```

# Get

bitcask.go

```go
func (b *Bitcask) get(key []byte) (internal.Entry, error) {
   var df data.Datafile
   //根据key查找索引信息
   value, found := b.trie.Search(key)
   if !found {
      return internal.Entry{}, ErrKeyNotFound
   }
   //强转
   item := value.(internal.Item)
   //获取所属文件
   if item.FileID == b.curr.FileID() {
      df = b.curr
   } else {
      df = b.datafiles[item.FileID]
   }
   //根据offset、size查找value
   e, err := df.ReadAt(item.Offset, item.Size)
   if err != nil {
      return internal.Entry{}, err
   }
   //设置了过期时间，判断是否过期
   if e.Expiry != nil && e.Expiry.Before(time.Now().UTC()) {
      //删除
      _ = b.delete(key)
      return internal.Entry{}, ErrKeyExpired
   }
  //检测数据是否损毁
   checksum := crc32.ChecksumIEEE(e.Value)
   if checksum != e.Checksum {
      return internal.Entry{}, ErrChecksumFailed
   }
   return e, nil
}
```

# delete

bitcask.go

```go
func (b *Bitcask) Delete(key []byte) error {
   b.mu.Lock()
   defer b.mu.Unlock()
   return b.delete(key)
}
```

```go
func (b *Bitcask) delete(key []byte) error {
  //追加空值
   _, _, err := b.put(key, []byte{}, Feature{})
   if err != nil {
      return err
   }
  //查找索引信息
   if item, found := b.trie.Search(key); found {
     //回收的空间大小
      b.metadata.ReclaimableSpace += item.(internal.Item).Size + codec.MetaInfoSize + int64(len(key))
   }
   //索引中删除
   b.trie.Delete(key)
   return nil
}
```

# Close

```go
func (b *Bitcask) Close() error {
   b.mu.RLock()
   defer func() {
      b.mu.RUnlock()
      b.Flock.Unlock()
   }()

   return b.close()
}
```

```go
func (b *Bitcask) close() error {
   if err := b.saveIndex(); err != nil { //保存索引
      return err
   }

   b.metadata.IndexUpToDate = true
   if err := b.saveMetadata(); err != nil { //保存metadata
      return err
   }

   for _, df := range b.datafiles {
      if err := df.Close(); err != nil { //关闭文件;非只读文件将数据刷新到磁盘
         return err
      }
   }

   return b.curr.Close()
}
```



# Write

```go
w, err = os.OpenFile(path, os.O_WRONLY|os.O_APPEND|os.O_CREATE, fileMode)
enc := codec.NewEncoder(w)
```

```go
func NewEncoder(w io.Writer) *Encoder {
   return &Encoder{w: bufio.NewWriter(w)}
}
```

```go
type Encoder struct {
   w *bufio.Writer
}
```

```go
func (e *Encoder) Encode(msg internal.Entry) (int64, error) {
  //创建byte数据存放key的长度和value的长度
   var bufKeyValue = make([]byte, keySize+valueSize)
  //将key的长度写入bufKeyValue
   binary.BigEndian.PutUint32(bufKeyValue[:keySize], uint32(len(msg.Key)))
   //将value的长度写入bufKeyValue
   binary.BigEndian.PutUint64(bufKeyValue[keySize:keySize+valueSize], uint64(len(msg.Value)))
   //将字节数组中的数据写入缓冲区
   if _, err := e.w.Write(bufKeyValue); err != nil {
      return 0, errors.Wrap(err, "failed writing key & value length prefix")
   }
	//将key的内容写入缓冲区
   if _, err := e.w.Write(msg.Key); err != nil {
      return 0, errors.Wrap(err, "failed writing key data")
   }
  //将value的内容写入缓冲区
   if _, err := e.w.Write(msg.Value); err != nil {
      return 0, errors.Wrap(err, "failed writing value data")
   }
	//截取4字节的数组存放校验码
   bufChecksumSize := bufKeyValue[:checksumSize]
  //将校验码写入字节数据
   binary.BigEndian.PutUint32(bufChecksumSize, msg.Checksum)
  //将校验字节码写入缓冲区
   if _, err := e.w.Write(bufChecksumSize); err != nil {
      return 0, errors.Wrap(err, "failed writing checksum data")
   }
	//截取8字节，存放过期时间
   bufTTL := bufKeyValue[:ttlSize]
   if msg.Expiry == nil {//将过期时间写入字节数组
      binary.BigEndian.PutUint64(bufTTL, uint64(0))
   } else {
      binary.BigEndian.PutUint64(bufTTL, uint64(msg.Expiry.Unix()))
   }
  //将过期时间写入缓冲区
   if _, err := e.w.Write(bufTTL); err != nil {
      return 0, errors.Wrap(err, "failed writing ttl data")
   }
	//调用内部writer的write方法
   if err := e.w.Flush(); err != nil {
      return 0, errors.Wrap(err, "failed flushing data")
   }
	//返回写入的长度
   return int64(keySize + valueSize + len(msg.Key) + len(msg.Value) + checksumSize + ttlSize), nil
}
```

# Read

```go
r, err = os.Open(path) //只读模式
dec := codec.NewDecoder(r, maxKeySize, maxValueSize)
```



```go
func NewDecoder(r io.Reader, maxKeySize uint32, maxValueSize uint64) *Decoder {
   return &Decoder{
      r:            r,
      maxKeySize:   maxKeySize,
      maxValueSize: maxValueSize,
   }
}
```



```go
type Decoder struct {
   r            io.Reader
   maxKeySize   uint32
   maxValueSize uint64
}
```



```go
func (d *Decoder) Decode(v *internal.Entry) (int64, error) { //顺序读
   if v == nil {
      return 0, errCantDecodeOnNilEntry
   }
	//读取字节，字节存放key的长度、value的长度
   prefixBuf := make([]byte, keySize+valueSize)
   _, err := io.ReadFull(d.r, prefixBuf)
   if err != nil {
      return 0, err
   }
	//获取key/value的长度
   actualKeySize, actualValueSize, err := getKeyValueSizes(prefixBuf, d.maxKeySize, d.maxValueSize)
   if err != nil {
      return 0, err
   }
	//buf用于存放读取的key内容、value内容、校验码、过期时间
   buf := make([]byte, uint64(actualKeySize)+actualValueSize+checksumSize+ttlSize)
   if _, err = io.ReadFull(d.r, buf); err != nil {
      return 0, errTruncatedData
   }

   decodeWithoutPrefix(buf, actualKeySize, v)
   return int64(keySize + valueSize + uint64(actualKeySize) + actualValueSize + checksumSize + ttlSize), nil
}
```

```go
func getKeyValueSizes(buf []byte, maxKeySize uint32, maxValueSize uint64) (uint32, uint64, error) {
  //读取key的长度
   actualKeySize := binary.BigEndian.Uint32(buf[:keySize])
  //读取value的长度
   actualValueSize := binary.BigEndian.Uint64(buf[keySize:])
		//验证key、value的合法性
   if (maxKeySize > 0 && actualKeySize > maxKeySize) || (maxValueSize > 0 && actualValueSize > maxValueSize) || actualKeySize == 0 {

      return 0, 0, errInvalidKeyOrValueSize
   }

   return actualKeySize, actualValueSize, nil
}
```

```go
func decodeWithoutPrefix(buf []byte, valueOffset uint32, v *internal.Entry) {
   v.Key = buf[:valueOffset]
   v.Value = buf[valueOffset : len(buf)-checksumSize-ttlSize]
   v.Checksum = binary.BigEndian.Uint32(buf[len(buf)-checksumSize-ttlSize : len(buf)-ttlSize])
   v.Expiry = getKeyExpiry(buf)
}
```

```go
func getKeyExpiry(buf []byte) *time.Time {
   expiry := binary.BigEndian.Uint64(buf[len(buf)-ttlSize:])
   if expiry == uint64(0) {
      return nil
   }
   t := time.Unix(int64(expiry), 0).UTC()
   return &t
}
```