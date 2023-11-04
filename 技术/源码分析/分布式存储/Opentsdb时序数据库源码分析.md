# OpenTsdb源码分析

* [写入数据](#写入数据)
  * [验证写入数据的合法性](#验证写入数据的合法性)
  * [值类型为长整形](#值类型为长整形)
  * [值类型为浮点形](#值类型为浮点形)
  * [写入数据](#写入数据-1)
    * [生成RowKey](#生成rowkey)
    * [构建列的限定符](#构建列的限定符)
* [读取数据](#读取数据)


# 写入数据

net.opentsdb.tsd.PutDataPointRpc#execute(net.opentsdb.core.TSDB, Channel, java.lang.String[])

```java
public Deferred<Object> execute(final TSDB tsdb, final Channel chan,
                                final String[] cmd) {
  requests.incrementAndGet();
  String errmsg = null;
  try {
    final class PutErrback implements Callback<Exception, Exception> {
      public Exception call(final Exception arg) {
        if (chan.isConnected()) {
          chan.write("put: HBase error: " + arg.getMessage() + '\n');
        }
        hbase_errors.incrementAndGet();
        return arg;
      }
      public String toString() {
        return "report error to channel";
      }
    }
    //写数据
    return importDataPoint(tsdb, cmd).addErrback(new PutErrback());
  } catch (NumberFormatException x) {
    errmsg = "put: invalid value: " + x.getMessage() + '\n';
    invalid_values.incrementAndGet();
  } catch (IllegalArgumentException x) {
    errmsg = "put: illegal argument: " + x.getMessage() + '\n';
    illegal_arguments.incrementAndGet();
  } catch (NoSuchUniqueName x) {
    errmsg = "put: unknown metric: " + x.getMessage() + '\n';
    unknown_metrics.incrementAndGet();
  }
  if (errmsg != null && chan.isConnected()) {
    chan.write(errmsg);
  }
  return Deferred.fromResult(null);
}
```

## 验证写入数据的合法性

net.opentsdb.tsd.PutDataPointRpc#importDataPoint

```java
private Deferred<Object> importDataPoint(final TSDB tsdb, final String[] words) {
  words[0] = null; // Ditch the "put".
  if (words.length < 5) {  // Need at least: metric timestamp value tag
    //               ^ 5 and not 4 because words[0] is "put".
    throw new IllegalArgumentException("not enough arguments"
                                       + " (need least 4, got " + (words.length - 1) + ')');
  }
  // [metric, timestamp, value, ..tags..]
  //监控指标
  final String metric = words[1];
  if (metric.length() <= 0) {
    throw new IllegalArgumentException("empty metric name");
  }
  //时间戳
  final long timestamp;
  if (words[2].contains(".")) {
    timestamp = Tags.parseLong(words[2].replace(".", "")); 
  } else {
    timestamp = Tags.parseLong(words[2]);
  }
  if (timestamp <= 0) {
    throw new IllegalArgumentException("invalid timestamp: " + timestamp);
  }
  //监控指标的值
  final String value = words[3];
  if (value.length() <= 0) {
    throw new IllegalArgumentException("empty value");
  }
  //解析组装tag
  final HashMap<String, String> tags = new HashMap<String, String>();
  for (int i = 4; i < words.length; i++) {
    if (!words[i].isEmpty()) {
      Tags.parse(tags, words[i]);
    }
  }
  if (Tags.looksLikeInteger(value)) { //整数
    return tsdb.addPoint(metric, timestamp, Tags.parseLong(value), tags);
  } else {  // 浮点数
    return tsdb.addPoint(metric, timestamp, Float.parseFloat(value), tags);
  }
}
```

## 值类型为长整形

net.opentsdb.core.TSDB#addPoint(java.lang.String, long, long, java.util.Map<java.lang.String,java.lang.String>)

```java
public Deferred<Object> addPoint(final String metric,
                                 final long timestamp,
                                 final long value,
                                 final Map<String, String> tags) {
   //尽量减少value占用的空间
  final byte[] v;
  if (Byte.MIN_VALUE <= value && value <= Byte.MAX_VALUE) {
    v = new byte[] { (byte) value };
  } else if (Short.MIN_VALUE <= value && value <= Short.MAX_VALUE) {
    v = Bytes.fromShort((short) value);
  } else if (Integer.MIN_VALUE <= value && value <= Integer.MAX_VALUE) {
    v = Bytes.fromInt((int) value);
  } else {
    v = Bytes.fromLong(value);
  }
  final short flags = (short) (v.length - 1);  // Just the length.
  return addPointInternal(metric, timestamp, v, tags, flags);
}
```

## 值类型为浮点形

net.opentsdb.core.TSDB#addPoint(java.lang.String, long, float, java.util.Map<java.lang.String,java.lang.String>)

```java
public Deferred<Object> addPoint(final String metric,
                                 final long timestamp,
                                 final float value,
                                 final Map<String, String> tags) {
  //验证值的合法性
  if (Float.isNaN(value) || Float.isInfinite(value)) {
    throw new IllegalArgumentException("value is NaN or Infinite: " + value
                                       + " for metric=" + metric
                                       + " timestamp=" + timestamp);
  }
  final short flags = Const.FLAG_FLOAT | 0x3;  // A float stored on 4 bytes.
  return addPointInternal(metric, timestamp,
                          Bytes.fromInt(Float.floatToRawIntBits(value)),
                          tags, flags);
}
```

## 写入数据

net.opentsdb.core.TSDB#addPointInternal

```java
private Deferred<Object> addPointInternal(final String metric,
                                          final long timestamp,
                                          final byte[] value,
                                          final Map<String, String> tags,
                                          final short flags) {
  //验证时间戳
  // we only accept positive unix epoch timestamps in seconds or milliseconds
  if (timestamp < 0 || ((timestamp & Const.SECOND_MASK) != 0 && 
      timestamp > 9999999999999L)) {
    throw new IllegalArgumentException((timestamp < 0 ? "negative " : "bad")
        + " timestamp=" + timestamp
        + " when trying to add value=" + Arrays.toString(value) + '/' + flags
        + " to metric=" + metric + ", tags=" + tags);
  }
  //检测监控指标和标签
  IncomingDataPoints.checkMetricAndTags(metric, tags);
  //生成rowkey
  final byte[] row = IncomingDataPoints.rowKeyTemplate(this, metric, tags);
  final long base_time;
  //构建列的限定符
  final byte[] qualifier = Internal.buildQualifier(timestamp, flags);
  //将时间戳换算成小时，填充到rowkey中
  if ((timestamp & Const.SECOND_MASK) != 0) {//将时间戳换算成秒再计算base_time
    // drop the ms timestamp to seconds to calculate the base timestamp
    base_time = ((timestamp / 1000) - 
        ((timestamp / 1000) % Const.MAX_TIMESPAN));
  } else {
    base_time = (timestamp - (timestamp % Const.MAX_TIMESPAN));
  }
  Bytes.setInt(row, (int) base_time, metrics.width());
  //合并
  scheduleForCompaction(row, (int) base_time);
  //创建写请求
  final PutRequest point = new PutRequest(table, row, FAMILY, qualifier, value);
  // TODO(tsuna): Add a callback to time the latency of HBase and store the
  // timing in a moving Histogram (once we have a class for this).
  Deferred<Object> result = client.put(point);
  if (!config.enable_realtime_ts() && !config.enable_tsuid_incrementing() && 
      !config.enable_tsuid_tracking() && rt_publisher == null) {
    return result;
  }
  
  final byte[] tsuid = UniqueId.getTSUIDFromKey(row, METRICS_WIDTH, 
      Const.TIMESTAMP_BYTES);
  
  // for busy TSDs we may only enable TSUID tracking, storing a 1 in the
  // counter field for a TSUID with the proper timestamp. If the user would
  // rather have TSUID incrementing enabled, that will trump the PUT
  if (config.enable_tsuid_tracking() && !config.enable_tsuid_incrementing()) {
    final PutRequest tracking = new PutRequest(meta_table, tsuid, 
        TSMeta.FAMILY(), TSMeta.COUNTER_QUALIFIER(), Bytes.fromLong(1));
    client.put(tracking);
  } else if (config.enable_tsuid_incrementing() || config.enable_realtime_ts()) {
    TSMeta.incrementAndGetCounter(TSDB.this, tsuid);
  }
  
  if (rt_publisher != null) {
    rt_publisher.sinkDataPoint(metric, timestamp, value, tags, tsuid, flags);
  }
  return result;
}
```

### 生成RowKey

net.opentsdb.core.IncomingDataPoints#rowKeyTemplate

```java
static byte[] rowKeyTemplate(final TSDB tsdb,
                             final String metric,
                             final Map<String, String> tags) {
  final short metric_width = tsdb.metrics.width();
  final short tag_name_width = tsdb.tag_names.width();
  final short tag_value_width = tsdb.tag_values.width();
  final short num_tags = (short) tags.size();
  //rowKey占用空间大小
  int row_size = (metric_width + Const.TIMESTAMP_BYTES
                  + tag_name_width * num_tags
                  + tag_value_width * num_tags);
  final byte[] row = new byte[row_size];
  //写入metricId的位置
  short pos = 0;
  //获取监控指标的唯一Id
  copyInRowKey(row, pos, (tsdb.config.auto_metric() ? 
      tsdb.metrics.getOrCreateId(metric) : tsdb.metrics.getId(metric)));
  pos += metric_width;
  //写入tag的位置
  pos += Const.TIMESTAMP_BYTES;
  //获取tagName、tagValue的唯一Id
  for(final byte[] tag : Tags.resolveOrCreateAll(tsdb, tags)) {
    copyInRowKey(row, pos, tag);
    pos += tag.length;
  }
  return row;
}
```

### 构建列的限定符

net.opentsdb.core.Internal#buildQualifier

```java
public static byte[] buildQualifier(final long timestamp, final short flags) {
  final long base_time;
  if ((timestamp & Const.SECOND_MASK) != 0) {
    // drop the ms timestamp to seconds to calculate the base timestamp
    base_time = ((timestamp / 1000) - ((timestamp / 1000) 
        % Const.MAX_TIMESPAN));
    final int qual = (int) (((timestamp - (base_time * 1000) 
        << (Const.MS_FLAG_BITS)) | flags) | Const.MS_FLAG);
    return Bytes.fromInt(qual);
  } else {
    base_time = (timestamp - (timestamp % Const.MAX_TIMESPAN));
    final short qual = (short) ((timestamp - base_time) << Const.FLAG_BITS
        | flags);
    return Bytes.fromShort(qual);
  }
}
```

# 读取数据

net.opentsdb.tsd.QueryRpc#execute

```java
public void execute(final TSDB tsdb, final HttpQuery query) 
  throws IOException {
  if (query.method() != HttpMethod.GET && query.method() != HttpMethod.POST) {
    throw new BadRequestException(HttpResponseStatus.METHOD_NOT_ALLOWED, 
        "Method not allowed", "The HTTP method [" + query.method().getName() +
        "] is not permitted for this endpoint");
  }
  final String[] uri = query.explodeAPIPath();
  final String endpoint = uri.length > 1 ? uri[1] : ""; 
  if (endpoint.toLowerCase().equals("last")) {
    handleLastDataPointQuery(tsdb, query);
    return;
  } else {
    handleQuery(tsdb, query);
  }
}
```
