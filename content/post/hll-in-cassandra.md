+++
date = "2016-12-04T18:00:00+03:00"
draft = false
title = "HLL counters as Cassandra user defined aggregates"
slug = "hll-in-cassandra"
+++
In [the previous article](../comparison-of-hll) we discussed usage of hll countes for finding the number of unique values in a collection. But in this article we will see how to use them in Cassandra db internally in order to provide such functionality using user defined functions(UDF) and aggregates(UDA).

<!--more-->
## Important

This approach can work **only with Cassandra 2.2.x.** In Cassandra 3.x there are more restrictions for UDF so it is impossible to use external libraries inside user functions. In the future may be there will be the way to turn off the restrictions(see [CASSANDRA-9892](https://issues.apache.org/jira/browse/CASSANDRA-9892)) but now it is impossible as well.

## Software

For this example I use:

* docker
* docker-machine
* docker-compose
* cassandra:2.2.8
* hll-library - [hyperloglog](https://github.com/prasanthj/hyperloglog)
* sbt
* Datastax DevCenter - for testing CQL queries. You can use `cqlsh` if it is more convenient for you

## Steps

* Install the necessary software
* Check out code of the example from here: [https://github.com/coffius/koffio-uda-hll](https://github.com/coffius/koffio-uda-hll)
* `sbt assembly` - build the jar file the hll
* `docker build` - build a docker image for Cassandra with our library inside
* `docker-compose up` - start the created Cassandra container inside docker
* Test hll counters - there are `hll_uda.cql` and `test_data.cql` with example queries for Cassandra

More information about how exactly it works you can find below.

## Simple example of UDA

At first let's take a look at how to define a very simple custom aggregate function:

```html
-- A simple example of creating functions for a user defined aggregate
CREATE OR REPLACE FUNCTION udf.avg_state ( state tuple<int,bigint>, val int ) CALLED ON NULL INPUT RETURNS tuple<int,bigint> LANGUAGE java AS
  'if (val !=null) { state.setInt(0, state.getInt(0)+1); state.setLong(1, state.getLong(1)+val.intValue()); } return state;';

CREATE OR REPLACE FUNCTION udf.avg_final ( state tuple<int,bigint> ) CALLED ON NULL INPUT RETURNS double LANGUAGE java AS
  'double r = 0; if (state.getInt(0) == 0) return null; r = state.getLong(1); r/= state.getInt(0); return Double.valueOf(r);';

CREATE AGGREGATE IF NOT EXISTS udf.avg ( int )
SFUNC avg_state STYPE tuple<int,bigint> FINALFUNC avg_final INITCOND (0,0);
```

For one aggregate we need two functions - a state function and a final function. The first accumulates the state and the second makes final calculation using the state from the first one.
More information about it you can find [here in official docs](https://docs.datastax.com/en/cql/3.3/cql/cql_using/useCreateUDA.html).

## Prepare HLL for Cassandra

The first thing that we have to do is to implement two functions that we can use in Cassandra aggregate. We cannot implement necessary logic directly in CQL-query like above because it would be too complex. For example, it is impossible to import classes right in CQL-code, so we need to use full names for classes like `io.koff.full.path.to.Clazz`.
So we do it using Java and then add the result jar into the Cassandra classpath in the docker container.

Example is here: [HLL.java](https://github.com/coffius/koffio-uda-hll/blob/master/src/main/java/io/koff/udf/HLL.java)

This is our state function:

```java
/**
 * State function. Aggregates values to hll counter
 * @param state byte buffer with serialized HLL counter
 * @param value the value to add to counter
 * @return hll counter with the new value
 */
public final static ByteBuffer stateFunc(ByteBuffer state, int value) {
	int capacity = state.capacity();
	// if it is the first call then state buffer should be empty
	if(capacity == 0) {
		HyperLogLog.HyperLogLogBuilder builder = new HyperLogLog.HyperLogLogBuilder();
		HyperLogLog hll = builder.build();
		hll.addInt(value);
		return serialize(hll);
	} else {
		// if there is data in state buffer then we need to use it
		HyperLogLog hll = deserialize(state);
		hll.addInt(value);
		return serialize(hll);
	}
}
```

And this is our final function:

```java
public final static long finalFunc(ByteBuffer state) {
	HyperLogLog hll = deserialize(state);
	return hll.count();
}
```

Important note - both functions should depend only on their parameters(see [pure functions](https://en.wikipedia.org/wiki/Pure_function))

## Deploy code

Next thing we should do is to add our code to Cassandra and enable UDF in Cassandra cofnig in order to make possible to use aggregates. 

At first we build a fat jar(jar with all dependencies) using [sbt-assembly](https://github.com/sbt/sbt-assembly) plugin.

in [build.sbt](https://github.com/coffius/koffio-uda-hll/blob/master/build.sbt):

```scala
assemblyJarName in assembly := "hll.jar"
// we do not need test classes in the result jar
test in assembly := {}
// define an output name of the result jar in the root folder. A bit easier for docker to find it there
target in assembly := new java.io.File(".")
//we do not need the scala library in the result jar
assemblyOption in assembly := (assemblyOption in assembly).value.copy(includeScala = false)
// compile java code with compatibility with JDK 7 which is used for Cassandra inside Docker
javacOptions ++= Seq("-source", "1.7", "-target", "1.7")
```

in [project/assembly.sbt](https://github.com/coffius/koffio-uda-hll/blob/master/project/assembly.sbt):

```scala
// use sbt-assembly for building a fat jar
addSbtPlugin("com.eed3si9n" % "sbt-assembly" % "0.14.3")
```
Then we add the jar file to the docker container along with [the patched Cassandra config](https://github.com/coffius/koffio-uda-hll/blob/master/cassandra.yaml#L879) where the flag `enable_user_defined_functions` is set to true:

[Dockerfile](https://github.com/coffius/koffio-uda-hll/blob/master/Dockerfile)

```html
FROM cassandra:2.2.8

# copy the cassandra config with the permission for user defined functions
COPY ./cassandra.yaml /etc/cassandra/cassandra.yaml

# copy the jar with out implementation of hll functions
COPY ./hll.jar /usr/share/cassandra/lib/
```

## Create UDA for HLL

The last step is to create UDA which will use our functions:

[hll_uda.cql](https://github.com/coffius/koffio-uda-hll/blob/master/hll_uda.cql)

```html
-- Keyspace definition
CREATE KEYSPACE IF NOT EXISTS udf
WITH replication = {
	'class' : 'SimpleStrategy',
	'replication_factor' : 1
};

-- Functions for hll counters
CREATE OR REPLACE FUNCTION udf.hll_state ( state blob, val int ) CALLED ON NULL INPUT RETURNS blob LANGUAGE java AS
  'return io.koff.udf.HLL.stateFunc(state, val);';

CREATE OR REPLACE FUNCTION udf.hll_final ( state blob ) CALLED ON NULL INPUT RETURNS bigint LANGUAGE java AS
  'return io.koff.udf.HLL.finalFunc(state);';

-- we need an empty ByteBuffer at the first step so we make it from the empty string
CREATE AGGREGATE IF NOT EXISTS udf.hll ( int ) SFUNC hll_state STYPE blob FINALFUNC hll_final INITCOND textAsBlob('');
```

We need to use full names for our functions.

## Test

Now we are ready to test it :)

Let's make a simple table where we will store events about user visits for different wares and add some data in it:

```html
CREATE TABLE IF NOT EXISTS udf.ware_hits (
	ware_id int,
	date_time timestamp,
	visitor_id int,
	PRIMARY KEY (ware_id, date_time)
);

INSERT INTO udf.ware_hits (ware_id, date_time, visitor_id) VALUES (1, toTimestamp(now()), 1);
INSERT INTO udf.ware_hits (ware_id, date_time, visitor_id) VALUES (1, toTimestamp(now()), 2);
INSERT INTO udf.ware_hits (ware_id, date_time, visitor_id) VALUES (1, toTimestamp(now()), 3);
INSERT INTO udf.ware_hits (ware_id, date_time, visitor_id) VALUES (1, toTimestamp(now()), 1);
INSERT INTO udf.ware_hits (ware_id, date_time, visitor_id) VALUES (1, toTimestamp(now()), 2);
INSERT INTO udf.ware_hits (ware_id, date_time, visitor_id) VALUES (1, toTimestamp(now()), 3);
INSERT INTO udf.ware_hits (ware_id, date_time, visitor_id) VALUES (1, toTimestamp(now()), 4);
INSERT INTO udf.ware_hits (ware_id, date_time, visitor_id) VALUES (1, toTimestamp(now()), 5);
INSERT INTO udf.ware_hits (ware_id, date_time, visitor_id) VALUES (1, toTimestamp(now()), 6);
```

So now there are 9 events for ware[id: 1] with 6 unique users. We can get this numbers using these queries:

```html
SELECT count(visitor_id) as total_count FROM udf.ware_hits WHERE ware_id = 1; --total_count = 9
SELECT udf.hll(visitor_id) as unique_users FROM udf.ware_hits WHERE ware_id = 1; --out aggregate returns unique_users = 6
```
## Afterword
It was shown how to implement custom aggregates for Cassandra by the example of hll counters. But before use this idea you also need to understand how aggregates work in Cassandra in the distributed environment.
This information you can find in the articles [here](http://www.doanduyhai.com/blog/?p=1876 "Cassandra UDF/UDA Technical Deep Dive") and [here](http://www.doanduyhai.com/blog/?p=2015 "Cassandra User Defined Aggregates in action: best practices and caveats"). Maybe Spark will be better choice in your case.

## Links

* [Source code for this article](https://github.com/coffius/koffio-uda-hll)
* [prasanthj/hyperloglog](https://github.com/prasanthj/hyperloglog)
* [Comparison of HLL implementations](../comparison-of-hll) - if you are interested to know about other HLL implementations
* [Cassandra UDF/UDA Technical Deep Dive](http://www.doanduyhai.com/blog/?p=1876) 
* [Cassandra User Defined Aggregates in action: best practices and caveats](http://www.doanduyhai.com/blog/?p=2015)