# 概述

Bitcask存储模型，写时顺序写，然后建立索引信息。索引放在内存中，通过Adaptive Radix Tree进行存储。

# 写数据

bitcask.go

```go
func (b *Bitcask) Put(key, value []byte, options ...PutOptions) error {
   //检测key、value是否合法
   if len(key) == 0 {
      return ErrEmptyKey
   }
   if b.config.MaxKeySize > 0 && uint32(len(key)) > b.config.MaxKeySize {
      return ErrKeyTooLarge
   }
   if b.config.MaxValueSize > 0 && uint64(len(value)) > b.config.MaxValueSize {
      return ErrValueTooLarge
   }
   var feature Feature
   for _, opt := range options {
      if err := opt(&feature); err != nil {
         return err
      }
   }
   b.mu.Lock()
   defer b.mu.Unlock()
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

# 读数据

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
   checksum := crc32.ChecksumIEEE(e.Value)
   if checksum != e.Checksum {
      return internal.Entry{}, ErrChecksumFailed
   }
   return e, nil
}
```

# 删除数据

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
      b.metadata.ReclaimableSpace += item.(internal.Item).Size + codec.MetaInfoSize + int64(len(key))
   }
   //索引中删除
   b.trie.Delete(key)
   return nil
}
```

