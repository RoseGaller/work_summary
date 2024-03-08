# 创建属性

com.baidu.hugegraph.schema.builder.PropertyKeyBuilder#create

```java
SchemaElement.checkName(this.name,
                        this.transaction.graph().configuration());
PropertyKey propertyKey = this.transaction.getPropertyKey(this.name); //判断是否已经存在
if (propertyKey != null) {
    if (this.checkExist) {
        throw new ExistedException("property key", this.name);
    }
    return propertyKey;
}
this.checkUserData(Action.INSERT);

propertyKey = this.build(); //构建属性信息
this.transaction.addPropertyKey(propertyKey);
return propertyKey;
```

com.baidu.hugegraph.schema.builder.PropertyKeyBuilder#build

```java
HugeGraph graph = this.transaction.graph();
Id id = this.transaction.getNextId(HugeType.PROPERTY_KEY); //获取属性的唯一标志
PropertyKey propertyKey = new PropertyKey(graph, id, this.name); //新建属性
propertyKey.dataType(this.dataType); //属性类型（string、int、float...）
propertyKey.cardinality(this.cardinality); //此属性值是single、list、set
for (Map.Entry<String, Object> entry : this.userdata.entrySet()) {
    propertyKey.userdata(entry.getKey(), entry.getValue());
}
return propertyKey;
```

com.baidu.hugegraph.backend.tx.SchemaTransaction#addPropertyKey

```java
this.addSchema(propertyKey);
```

com.baidu.hugegraph.backend.tx.SchemaTransaction#addSchema

```java
this.beforeWrite();
this.doInsert(this.serialize(schema));//序列化为BackendEntry
this.indexTx.updateNameIndex(schema, false); //更新索引信息
this.afterWrite();
```

com.baidu.hugegraph.backend.tx.SchemaIndexTransaction#updateNameIndex

```java
if (!this.needIndexForName()) {
    return;
}

IndexLabel indexLabel = IndexLabel.label(element.type());
// Update name index if backend store not supports name-query
HugeIndex index = new HugeIndex(indexLabel);
index.fieldValues(element.name());
index.elementIds(element.id());

if (removed) {
    this.doEliminate(this.serializer.writeIndex(index));
} else {
    this.doAppend(this.serializer.writeIndex(index));
}
```

# 创建vertexLabel

```java
schema.vertexLabel("person") //顶点标签
      .properties("name", "age", "city") //包含的属性
      .primaryKeys("name")//设置主键
      .create();
```

com.baidu.hugegraph.schema.builder.VertexLabelBuilder#create

```java
SchemaElement.checkName(this.name,
                        this.transaction.graph().configuration());
VertexLabel vertexLabel = this.transaction.getVertexLabel(this.name); //判断顶点是否已经存在
if (vertexLabel != null) {
    if (this.checkExist) {
        throw new ExistedException("vertex label", this.name);
    }
    return vertexLabel;
}

this.checkProperties(Action.INSERT); //检查属性是否已经存在
this.checkIdStrategy(); //检查Id策略，用于生成顶点的唯一标志
this.checkNullableKeys(Action.INSERT);
this.checkUserData(Action.INSERT);

vertexLabel = this.build();
this.transaction.addVertexLabel(vertexLabel);
return vertexLabel;
```

com.baidu.hugegraph.schema.builder.VertexLabelBuilder#build

```java
HugeGraph graph = this.transaction.graph(); 
Id id = this.transaction.getNextId(HugeType.VERTEX_LABEL);//有存储引擎生成唯一Id
VertexLabel vertexLabel = new VertexLabel(graph, id, this.name); //填充顶点的信息
vertexLabel.idStrategy(this.idStrategy);
vertexLabel.enableLabelIndex(this.enableLabelIndex == null ||
                             this.enableLabelIndex);

for (String key : this.properties) {
    PropertyKey propertyKey = this.transaction.getPropertyKey(key);
    vertexLabel.property(propertyKey.id()); //属性Id
}
for (String key : this.primaryKeys) {
    PropertyKey propertyKey = this.transaction.getPropertyKey(key);
    vertexLabel.primaryKey(propertyKey.id()); //主键Id
}
for (String key : this.nullableKeys) {
    PropertyKey propertyKey = this.transaction.getPropertyKey(key);
    vertexLabel.nullableKey(propertyKey.id()); //不能为空的属性的Id
}
for (Map.Entry<String, Object> entry : this.userdata.entrySet()) {
    vertexLabel.userdata(entry.getKey(), entry.getValue());
}
return vertexLabel;
```

# 创建edgeLabel

```java
schema.edgeLabel("rated") //边的标签名
      .sourceLabel("reviewer")//源顶点标签名
  		.targetLabel("recipe")  //目标顶点标签名
      .create();
```

```java
SchemaElement.checkName(this.name,
                        this.transaction.graph().configuration());

EdgeLabel edgeLabel = this.transaction.getEdgeLabel(this.name); //判重
if (edgeLabel != null) {
    if (this.checkExist) {
        throw new ExistedException("edge label", this.name);
    }
    return edgeLabel;
}

if (this.frequency == Frequency.DEFAULT) {
    this.frequency = Frequency.SINGLE;
}
// These methods will check params and fill to member variables
this.checkRelation(); //检测sourceLabel、targetLabel不相同，并且都已存在
this.checkProperties(Action.INSERT); //检测边的属性都已经存在
this.checkSortKeys();
this.checkNullableKeys(Action.INSERT);
this.checkNullableKeys(Action.INSERT);

edgeLabel = this.build();
this.transaction.addEdgeLabel(edgeLabel);
return edgeLabel;
```

com.baidu.hugegraph.schema.builder.EdgeLabelBuilder#build

```java
HugeGraph graph = this.transaction.graph();
Id id = this.transaction.getNextId(HugeType.EDGE_LABEL);//生成唯一Id
EdgeLabel edgeLabel = new EdgeLabel(graph, id, this.name); //创建EdgeLabel
edgeLabel.sourceLabel(this.transaction.getVertexLabel(
                      this.sourceLabel).id()); //设置源顶点标签Id
edgeLabel.targetLabel(this.transaction.getVertexLabel(
                      this.targetLabel).id()); //设置目标顶点标签Id
edgeLabel.frequency(this.frequency);
edgeLabel.enableLabelIndex(this.enableLabelIndex == null ||
                           this.enableLabelIndex);
for (String key : this.properties) {
    PropertyKey propertyKey = this.transaction.getPropertyKey(key);
    edgeLabel.property(propertyKey.id()); //边的属性Id
}
for (String key : this.sortKeys) {
    PropertyKey propertyKey = this.transaction.getPropertyKey(key);
    edgeLabel.sortKey(propertyKey.id()); //排序属性Id
}
for (String key : this.nullableKeys) {
    PropertyKey propertyKey = this.transaction.getPropertyKey(key);
    edgeLabel.nullableKey(propertyKey.id()); //不为空属性Id
}
for (Map.Entry<String, Object> entry : this.userdata.entrySet()) {
    edgeLabel.userdata(entry.getKey(), entry.getValue());
}
return edgeLabel;
```

# 添加顶点

```java
GraphTransaction tx = graph.openTransaction();
Vertex james = tx.addVertex(T.label, "author", //指定顶点的标签名
                            "id", 1, "name", "James Gosling",
                            "age", 62, "lived", "Canadian"); //属性信息
```

com.baidu.hugegraph.HugeGraph#addVertex

```java
public Vertex addVertex(Object... keyValues) {
    return this.graphTransaction().addVertex(keyValues);
}
```

```java
public Vertex addVertex(Object... keyValues) {
    HugeElement.ElementKeys elemKeys = HugeElement.classifyKeys(keyValues);
    //根据label获取VertexLabel
    VertexLabel vertexLabel = this.checkVertexLabel(elemKeys.label());
    Id id = HugeVertex.getIdValue(elemKeys.id());
    //获取属性对应的Id
    List<Id> keys = this.graph().mapPkName2Id(elemKeys.keys());

    // Check whether id match with id strategy
    this.checkId(id, keys, vertexLabel);

    // Check whether passed all non-null property
    this.checkNonnullProperty(keys, vertexLabel);

    // 创建HugeVertex
    HugeVertex vertex = new HugeVertex(this, null, vertexLabel);

    // 设置属性值
    ElementHelper.attachProperties(vertex, keyValues);

    // 生成顶点Id
    if (this.graph().restoring() &&
        vertexLabel.idStrategy() == IdStrategy.AUTOMATIC) {
        // Resume id for AUTOMATIC id strategy in restoring mode
        vertex.assignId(id, true);
    } else {
        vertex.assignId(id);
    }
		//保存顶点
    return this.addVertex(vertex);
}
```

# 添加边

```java
Vertex james = tx.addVertex(T.label, "author",
                            "id", 1, "name", "James Gosling",
                            "age", 62, "lived", "Canadian"); //添加顶点
Vertex book2 = tx.addVertex(T.label, "book", "name", "java-2"); //添加顶点
james.addEdge("authored", book2, "contribution", "2017-4-28");//添加边

```

