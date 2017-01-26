---
title: Gremlin Neo4j Driver Benchmarks
layout: post
---


# Intro

Neo4j Embedded Vs. Bolt (remote) driver Vs. Gremlin Server (remote, embedded)

# Configurations

Hardware: Intel(R) Core(TM) i7-4900MQ CPU @ 2.80GHz, 8 cores, 32GB RAM, 500GB SSD, ext4 FS

Neo4j embedded

- gremlin-console
- client is gremlin
- neo4j-tinkerpop-api-impl v0.4-3.0.8
- Neo4j v3.0.8
- Gremlin v3.2.3
- :install org.neo4j neo4j-tinkerpop-api-impl 0.4-3.0.8

Neo4j bolt

- [neo4j-gremlin-bolt v0.2.16](https://github.com/SteelBridgeLabs/neo4j-gremlin-bolt)
- Neo4j remote
- client is bolt driver
- bolt v1.1.0
- gremlin v3.2.3
- :install com.steelbridgelabs.oss neo4j-gremlin-bolt 0.2.16

gremlin-server 3.2.3

- Gremlin server remote
- Neo4j embedded in server
- Neo4j v3.0.8
- Gremlin v3.2.3
- -i org.neo4j neo4j-tinkerpop-api-impl 0.4-3.0.8
- -i org.apache.tinkerpop neo4j-gremlin 3.2.3

# Setup

Using gremlin-console, a new neo4j database is created and loaded with the Grateful Dead graph data.

```groovy
import org.apache.tinkerpop.gremlin.neo4j.structure.*

// create a new graph
graph = Neo4jGraph.open('./graph.db')

// load graph
graph.io(graphml()).readGraph('data/grateful-dead.xml')

graph.close()

```

This graph is copied to both Neo4j standalone server and gremlin-server so each test environment has the same starting data and database.

All tests executed from gremlin-console.

# Embedded Test Setup

```groovy
import org.apache.tinkerpop.gremlin.neo4j.structure.*
graph = Neo4jGraph.open('./graph.db')
g = graph.traversal()
```

# Bolt Test Setup

```groovy
import com.steelbridgelabs.oss.neo4j.structure.*;
import org.neo4j.driver.v1.Driver;
import org.neo4j.driver.v1.GraphDatabase;
import org.neo4j.driver.v1.AuthTokens;
import com.steelbridgelabs.oss.neo4j.structure.providers.*;

// connect
driver = GraphDatabase.driver("bolt://localhost", AuthTokens.basic("neo4j", "notneo4j"));

vertexIdProvider = new Neo4JNativeElementIdProvider();
edgeIdProvider = new Neo4JNativeElementIdProvider();

graph = new Neo4JGraph(driver, vertexIdProvider, edgeIdProvider)
g = graph.traversal()
```

# Gremlin Server Setup

```groovy
graph = EmptyGraph.instance()
g = graph.traversal().withRemote('conf/remote-graph.properties')
```

# Benchmarks

Benchmark traversals were copied from gremlin-benchmark.  Instead of toList(), this test uses toStream() to minimize effects of memory/GC on the client.  

```groovy
batch=10

// 1
clockWithResult(batch){g.V().outE().inV().outE().inV().outE().inV().toStream().count()}

// 2
clockWithResult(batch){g.V().out().out().out().toStream().count()}

// 3
clockWithResult(batch){g.V().out().out().out().path().toStream().count()}

// 4
clockWithResult(batch){g.V().repeat(out()).times(2).toStream().count()}

// 5
clockWithResult(batch){g.V().repeat(out()).times(3).toStream().count()}

// 6
clockWithResult(batch){g.V().local(out().out().values("name").fold()).toStream().count()}

// 7
clockWithResult(batch){g.V().out().local(out().out().values("name").fold()).toStream().count()}

// 8
clockWithResult(batch){g.V().label().groupCount().toStream().count()}

// 9
clockWithResult(batch){g.V().match(
            __.as("a").has("name", "Garcia"),
            __.as("a").in("writtenBy").as("b"),
            __.as("a").in("sungBy").as("b")).select("b").values("name").toStream().count()}

// 10
clockWithResult(batch){g.E().hasLabel("writtenBy").where(__.inV().inE("sungBy").count().is(0)).subgraph("sg").toStream().count()}

```


# Benchmarks 2

These set of tests use valueMap(true) instead of returning a simple vertex.  This is to put the gremlin client on par with embedded clients.

```groovy
batch=10

// 11
clockWithResult(batch){g.V().outE().inV().outE().inV().outE().inV().valueMap(true).toStream().count()}

// 12
clockWithResult(batch){g.V().out().out().out().valueMap(true).toStream().count()}

// 13
//clockWithResult(batch){g.V().out().out().out().path().by(valueMap(true)).toStream().count()}

// 14
clockWithResult(batch){g.V().repeat(out()).times(2).valueMap(true).toStream().count()}

// 15
clockWithResult(batch){g.V().repeat(out()).times(3).valueMap(true).toStream().count()}

// 16
clockWithResult(batch){g.E().hasLabel("writtenBy").where(__.inV().inE("sungBy").count().is(0)).subgraph("sg").valueMap(true).toStream().count()}
```



# Results

clock() times are execution time averages in milliseconds.  

|Test|Embedded Neo4j|Gremlin Bolt|Gremlin Server|Result Count|
|---:|---:|---:|---:|---:|
|1|191.30|397.18|68.68|14,465,066|
|2|188.59|290.67|55.38|14,465,066|
|3|3,800.19|102,680.21|59,644.28|14,465,066|
|4|12.04|179.06|7.81|327,370|
|5|323.65|316.89|55.19|14,465,066|
|6|143.78|2,421.60|Error\*||
|7|242.84|2,556.07|Error\*||
|8|3.16|0.84|4.79|1|
|9|5.30|7.16|11.66|2|
|10|17.31|15.20|22.25|280|
|11|157.14|381.41|61.00|14,465,066|
|12|153.05|299.05|55.28|14,465,066|
|\*\*13|130,830.00|174,367.00|384,730.00||
|14|18.23|184.17|12.03|327,370|
|15|192.34|291.34|55.88|14,465,066|
|16|15.77|16.44|11.63|280|




\* Max frame length of 65536 has been exceeded

\*\* Only one execution due to time constraints


# Other Gremlin Bolt Issues

Attempting load a graph with either Neo4jNative or DatabaseSequence ID Providers results in:

```
gremlin> graph.io(graphml()).readGraph('data/grateful-dead.xml')
Property value [5] is of type class java.lang.Integer is not supported
Type ':help' or ':h' for help.
Display stack trace? [yN]y
java.lang.IllegalArgumentException: Property value [5] is of type class java.lang.Integer is not supported
 at org.apache.tinkerpop.gremlin.structure.Property$Exceptions.dataTypeOfPropertyValueNotSupported(Property.java:163)
 at org.apache.tinkerpop.gremlin.structure.Property$Exceptions.dataTypeOfPropertyValueNotSupported(Property.java:159)
 at com.steelbridgelabs.oss.neo4j.structure.Neo4JBoltSupport.checkPropertyValue(Neo4JBoltSupport.java:39)
 at com.steelbridgelabs.oss.neo4j.structure.Neo4JVertex.property(Neo4JVertex.java:728)
 at org.apache.tinkerpop.gremlin.structure.util.ElementHelper.attachProperties(ElementHelper.java:301)
 at com.steelbridgelabs.oss.neo4j.structure.Neo4JSession.addVertex(Neo4JSession.java:233)
 at com.steelbridgelabs.oss.neo4j.structure.Neo4JGraph.addVertex(Neo4JGraph.java:218)
 at org.apache.tinkerpop.gremlin.structure.io.graphml.GraphMLReader.findOrCreate(GraphMLReader.java:310)
```

When graph is loaded with neo4j-gremlin and then accessed using the gremlin-bolt driver with DatabaseSequence provider, it resulted in errors. Apparently it's trying to convert the schema's id, which does not exist, to a long.

```
gremlin> g.V()
Cannot coerce NULL to Java long
```
