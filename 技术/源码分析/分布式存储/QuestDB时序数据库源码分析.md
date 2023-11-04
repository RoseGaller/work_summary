# QuickStart

QuestDB是一个面向列的关系数据库，专为时间序列数据设计

```java
public class AppendRawTimeSeries {

    public static void main(String[] args) throws JournalException, ParserException {
        if (args.length < 1) {
            System.out.println("Usage: AppendRawTimeSeries <path>");
            System.exit(1);
        }
        final String location = args[0];
      //创建JournalFactory，生产Reader/Writer的工厂
        try (JournalFactory factory = new JournalFactory(location)) { 
            try (JournalWriter writer = factory.writer(
                    new JournalStructure("customers"). //存放数据的目录
                            $int("id"). //int类型的字段
                            $str("name"). //string类型的字段
                            $ts("updateDate"). //时间戳类型的字段
                            $())) { //先创建JournalMetadata再创建JournalWriter

                Rnd rnd = new Rnd();

                long timestamp = System.currentTimeMillis();
                for (int i = 0; i < 1000000; i++) {

                    // enforce timestamp order
                    JournalEntryWriter ew = writer.entryWriter(timestamp);

                    // columns accessed by index
                    ew.putInt(0, rnd.nextPositiveInt());
                    ew.putStr(1, rnd.nextChars(25));

                    // increment timestamp by 30 seconds
                    timestamp += 30000;

                    // append record to journal
                    ew.append();
                }
                // commit all records at once
                // there is no limit on how many records can be in the same transaction
                writer.commit();
            }
        }
    }
}
```

创建JournalStructure

com.questdb.factory.configuration.JournalStructure#JournalStructure(java.lang.String)

```java
public JournalStructure(String location) {
    this.location = location; //指定存储目录
}
```

com.questdb.factory.configuration.JournalStructure#$int

```java
public GenericIndexedBuilder $int(String name) {
    return new GenericIndexedBuilder(this, newMeta(name), ColumnType.INT, 4);//创建int类型的字段（类型、占用的空间）
}
```

com.questdb.factory.configuration.JournalStructure#newMeta

```java
private ColumnMetadata newMeta(String name) {
    int index = nameToIndexMap.get(name);
    if (index == -1) {
        ColumnMetadata meta = new ColumnMetadata().setName(name);//创建列的元数据信息
        metadata.add(meta);
        nameToIndexMap.put(name, metadata.size() - 1);
        return meta;
    } else {
        throw new JournalConfigurationException("Duplicate column: " + name);
    }
}
```

# References

https://questdb.io/