---
title: Storm Cassandra 集成
layout: documentation
documentation: true
---

### Apache Cassandra 的 Bolt API 实现

这个库提供了 Apache Cassandra 之上的核心 storm bolt .
提供简单的 DSL 来 map storm *Tuple* 到  Cassandra Query Language *Statement* （Cassandra 查询语言 *Statement*）.


### Configuration （配置）
以下属性可能会传递给 storm 配置.

| **Property name（属性名称）**                            | **Description（描述）** | **Default（默认）**         |
| ---------------------------------------------| ----------------| --------------------|
| **cassandra.keyspace**                       | -               |                     |
| **cassandra.nodes**                          | -               | {"localhost"}       |
| **cassandra.username**                       | -               | -                   |
| **cassandra.password**                       | -               | -                   |
| **cassandra.port**                           | -               | 9092                |
| **cassandra.output.consistencyLevel**        | -               | ONE                 |
| **cassandra.batch.size.rows**                | -               | 100                 |
| **cassandra.retryPolicy**                    | -               | DefaultRetryPolicy  |
| **cassandra.reconnectionPolicy.baseDelayMs** | -               | 100 (ms)            |
| **cassandra.reconnectionPolicy.maxDelayMs**  | -               | 60000 (ms)          |

### CassandraWriterBolt

#### Static import
```java

import static org.apache.storm.cassandra.DynamicStatementBuilder.*

```

#### Insert Query Builder （插入查询生成器）
##### Insert query including only the specified tuple fields （插入仅包含指定 tuple 字段的查询）.
```java

    new CassandraWriterBolt(
        async(
            simpleQuery("INSERT INTO album (title,year,performer,genre,tracks) VALUES (?, ?, ?, ?, ?);")
                .with(
                    fields("title", "year", "performer", "genre", "tracks")
                 )
            )
    );
```

##### Insert query including all tuple fields（插入包含所有 tuple 字段的查询）.
```java

    new CassandraWriterBolt(
        async(
            simpleQuery("INSERT INTO album (title,year,performer,genre,tracks) VALUES (?, ?, ?, ?, ?);")
                .with( all() )
            )
    );
```

##### Insert multiple queries from one input tuple （从一个 input tuple 插入多个查询）.
```java

    new CassandraWriterBolt(
        async(
            simpleQuery("INSERT INTO titles_per_album (title,year,performer,genre,tracks) VALUES (?, ?, ?, ?, ?);").with(all())),
            simpleQuery("INSERT INTO titles_per_performer (title,year,performer,genre,tracks) VALUES (?, ?, ?, ?, ?);").with(all()))
        )
    );
```

##### Insert query using QueryBuilder（使用 QueryBuilder 插入查询）
```java

    new CassandraWriterBolt(
        async(
            simpleQuery("INSERT INTO album (title,year,perfomer,genre,tracks) VALUES (?, ?, ?, ?, ?);")
                .with(all()))
            )
    )
```

##### Insert query with static bound query （使用 static bound query 插入 查询）
```java

    new CassandraWriterBolt(
         async(
            boundQuery("INSERT INTO album (title,year,performer,genre,tracks) VALUES (?, ?, ?, ?, ?);")
                .bind(all());
         )
    );
```

##### Insert query with static bound query using named setters and aliases （使用 named setters 和 aliases 插入带有 static bound query 的查询）
```java

    new CassandraWriterBolt(
         async(
            boundQuery("INSERT INTO album (title,year,performer,genre,tracks) VALUES (:ti, :ye, :pe, :ge, :tr);")
                .bind(
                    field("ti"),as("title"),
                    field("ye").as("year")),
                    field("pe").as("performer")),
                    field("ge").as("genre")),
                    field("tr").as("tracks"))
                ).byNamedSetters()
         )
    );
```

##### Insert query with bound statement load from storm configuration （从 storm 配置插入 bound statement load 的查询）
```java

    new CassandraWriterBolt(
         boundQuery(named("insertIntoAlbum"))
            .bind(all());
```

##### Insert query with bound statement load from tuple field （从 tuple 字段插入 bound statement load 的查询）
```java

    new CassandraWriterBolt(
         boundQuery(namedByField("cql"))
            .bind(all());
```

##### Insert query with batch statement （使用 batch 语句插入查询）
```java

    // Logged
    new CassandraWriterBolt(loggedBatch(
            simpleQuery("INSERT INTO titles_per_album (title,year,performer,genre,tracks) VALUES (?, ?, ?, ?, ?);").with(all())),
            simpleQuery("INSERT INTO titles_per_performer (title,year,performer,genre,tracks) VALUES (?, ?, ?, ?, ?);").with(all()))
        )
    );
// UnLogged
    new CassandraWriterBolt(unLoggedBatch(
            simpleQuery("INSERT INTO titles_per_album (title,year,performer,genre,tracks) VALUES (?, ?, ?, ?, ?);").with(all())),
            simpleQuery("INSERT INTO titles_per_performer (title,year,performer,genre,tracks) VALUES (?, ?, ?, ?, ?);").with(all()))
        )
    );
```

### 如何处理 query execution results （查询执行结果）

*ExecutionResultHandler* 接口可用于自定义 execution result （执行结果）应如何处理.

```java
public interface ExecutionResultHandler extends Serializable {
    void onQueryValidationException(QueryValidationException e, OutputCollector collector, Tuple tuple);

    void onReadTimeoutException(ReadTimeoutException e, OutputCollector collector, Tuple tuple);

    void onWriteTimeoutException(WriteTimeoutException e, OutputCollector collector, Tuple tuple);

    void onUnavailableException(UnavailableException e, OutputCollector collector, Tuple tuple);

    void onQuerySuccess(OutputCollector collector, Tuple tuple);
}
```

默认情况下,  CassandraBolt 将在所有的 Cassandra Exception 中 fails（失败）一个 tuple （请参阅 [BaseExecutionResultHandler](https://github.com/apache/storm/tree/master/external/storm-cassandra/blob/master/src/main/java/org/apache/storm/cassandra/BaseExecutionResultHandler.java)）.

```java
    new CassandraWriterBolt(insertInto("album").values(with(all()).build())
            .withResultHandler(new MyCustomResultHandler());
```

### Declare Output fields （声明输出字段）

CassandraBolt 可以声明 declare output fields  / stream output fields（输出字段/流输出字段）.
例如, 这可以用于在 error （错误）或者 chain queries （链式查询）上 remit （传递）一个 new tuple （新的元组）.

```java
    new CassandraWriterBolt(insertInto("album").values(withFields(all()).build())
            .withResultHandler(new EmitOnDriverExceptionResultHandler());
            .withStreamOutputFields("stream_error", new Fields("message");

    public static class EmitOnDriverExceptionResultHandler extends BaseExecutionResultHandler {
        @Override
        protected void onDriverException(DriverException e, OutputCollector collector, Tuple tuple) {
            LOG.error("An error occurred while executing cassandra statement", e);
            collector.emit("stream_error", new Values(e.getMessage()));
            collector.ack(tuple);
        }
    }
```

### Murmur3FieldGrouping

[Murmur3StreamGrouping](https://github.com/apache/storm/tree/master/external/storm-cassandra/blob/master/src/main/java/org/apache/storm/cassandra/Murmur3StreamGrouping.java) 可以用来优化 cassandra writes （cassandra 的写入）.
根据 specified row partition keys （指定的行分区键）, 该 stream 在 bolt 的 task 之间进行 partitioned （分区）.

```java
CassandraWriterBolt bolt = new CassandraWriterBolt(
    insertInto("album")
        .values(
            with(fields("title", "year", "performer", "genre", "tracks")
            ).build());
builder.setBolt("BOLT_WRITER", bolt, 4)
        .customGrouping("spout", new Murmur3StreamGrouping("title"))
```

### Trident API 支持
storm-cassandra 支持 用于将 data `inserting（插入）` Cassandra 的 Trident `state` API .
```java
        CassandraState.Options options = new CassandraState.Options(new CassandraContext());
        CQLStatementTupleMapper insertTemperatureValues = boundQuery(
                "INSERT INTO weather.temperature(weather_station_id, weather_station_name, event_time, temperature) VALUES(?, ?, ?, ?)")
                .bind(with(field("weather_station_id"), field("name").as("weather_station_name"), field("event_time").now(), field("temperature")));
        options.withCQLStatementTupleMapper(insertTemperatureValues);
        CassandraStateFactory insertValuesStateFactory =  new CassandraStateFactory(options);
        TridentState selectState = topology.newStaticState(selectWeatherStationStateFactory);
        stream = stream.stateQuery(selectState, new Fields("weather_station_id"), new CassandraQuery(), new Fields("name"));
        stream = stream.each(new Fields("name"), new PrintFunction(), new Fields("name_x"));
        stream.partitionPersist(insertValuesStateFactory, new Fields("weather_station_id", "name", "event_time", "temperature"), new CassandraStateUpdater(), new Fields());
```

以下 `state` API 用于从 Cassandra `querying（查询）` 数据.
```java
        CassandraState.Options options = new CassandraState.Options(new CassandraContext());
        CQLStatementTupleMapper insertTemperatureValues = boundQuery("SELECT name FROM weather.station WHERE id = ?")
                 .bind(with(field("weather_station_id").as("id")));
        options.withCQLStatementTupleMapper(insertTemperatureValues);
        options.withCQLResultSetValuesMapper(new TridentResultSetValuesMapper(new Fields("name")));
        CassandraStateFactory selectWeatherStationStateFactory =  new CassandraStateFactory(options);
        CassandraStateFactory selectWeatherStationStateFactory = getSelectWeatherStationStateFactory();
        TridentState selectState = topology.newStaticState(selectWeatherStationStateFactory);
        stream = stream.stateQuery(selectState, new Fields("weather_station_id"), new CassandraQuery(), new Fields("name"));         
```
