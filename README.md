Apache Storm - Elasticsearch
----------------------------

[Apache Storm](https://storm.apache.org/) is a free and open source distributed realtime computation system.

### Bolt/Trident API implementation for [Elasticsearch](https://www.elastic.co/)

This library provides core storm bolt and implements a Trident state on top of Elasticsearch.
It supports non-transactional, transactional, and opaque state types.

### Maven dependency
```xml
    <dependency>
      <groupId>com.github.fhuss</groupId>
      <artifactId>storm-elasticsearch</artifactId>
      <version>0.3.0</version>
    </dependency>
```
### TupleMapper / TridentTupleMapper
To index documents into elasticsearch you need to provide an implementation of following interfaces according to
you use bolt or trident state.

These two interfaces have only one method defined used to map tuple fields to a [Document](https://github.com/fhussonnois/storm-trident-elasticsearch/blob/master/src/main/java/com/github/fhuss/storm/elasticsearch/Document.java).

```java
    public interface TupleMapper<T> extends Serializable {
       T map(Tuple input);
    }
```

```java
    public interface TridentTupleMapper<T> extends Serializable {
       T map(TridentTuple input);
    }
```

To be indexed, a document requires at least following attributes:

- The **name** of the index
- The **type** of document
- The **source** document
    
For general use cases, a default implementation is provided [DefaultTupleMapper](https://github.com/fhussonnois/storm-trident-elasticsearch/blob/master/src/main/java/com/github/fhuss/storm/elasticsearch/mapper/impl/DefaultTupleMapper.java).

### Core Bolt / IndexBatchBolt
The IndexBatchBolt implementation relies on storm tick tuple feature and Elasticsearch [Bulk API](http://www.elasticsearch.org/guide/en/elasticsearch/reference/current/docs-bulk.html) to 
index many tuples.

### Trident State examples
#### Persistent Aggregate

```java
    FixedBatchSpout spout = new FixedBatchSpout(new Fields("sentence"), 3,
                new Values("the cow jumped over the moon"),
                new Values("the man went to the store and bought some candy"),
                new Values("four score and seven years ago"),
                new Values("how many apples can you eat"),
                new Values("to be or not to be the person"));
    spout.setCycle(true);

    TridentTopology topology = new TridentTopology();

    Settings settings = ImmutableSettings.settingsBuilder().loadFromClasspath("elasticsearch.yml").build();
    StateFactory stateFactory = ESIndexMapState.nonTransactional(new ClientFactory.LocalTransport(settings.getAsMap()), Tweet.class);
        
    topology.newStream("tweets", spout)
            .each(new Fields("sentence"), new DocumentBuilder(), new Fields("document"))
            .each(new Fields("document"), new ExtractDocumentInfo(), new Fields("id", "index", "type"))
            .groupBy(new Fields("index", "type", "id"))
            .persistentAggregate(stateFactory, new Fields("document"), new TweetBuilder(), new Fields("tweet"))
            .parallelismHint(1);
```

#### Search query using DRPC
```java
    TridentTopology topology = new TridentTopology();

    Settings settings = ImmutableSettings.settingsBuilder().loadFromClasspath("elasticsearch.yml").build();
    TridentState staticState = topology.newStaticState(new ESIndexState.Factory<>(new LocalTransport(settings.getAsMap()), Tweet.class));
    topology.newDRPCStream("search", drpc)
            .each(new Fields("args"), new ExtractSearchArgs(), new Fields("query", "indices", "types"))
            .groupBy(new Fields("query", "indices", "types"))
            .stateQuery(staticState, new Fields("query", "indices", "types"), new QuerySearchIndexQuery(), new Fields("tweet"))
            .each(new Fields("tweet"), new FilterNull())
            .each(new Fields("tweet"), new CreateJson(), new Fields("json"))
            .project(new Fields("json"));
```

## License

Licensed to the Apache Software Foundation (ASF) under one
or more contributor license agreements.  See the NOTICE file
distributed with this work for additional information
regarding copyright ownership.  The ASF licenses this file
to you under the Apache License, Version 2.0 (the
"License"); you may not use this file except in compliance
with the License.  You may obtain a copy of the License at

  http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing,
software distributed under the License is distributed on an
"AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
KIND, either express or implied.  See the License for the
specific language governing permissions and limitations
under the License.
