+++
date = "2015-09-22T22:01:13+03:00"
draft = false
title = "Comparison of HLL implementations"
slug = "comparison-of-hll"
+++

In this article we will look at HLL algorithm and different implementations of it.

## General Info

HLL is a propabalistic algorithm which is used for a estimation of unique values. 
More details about HLL you can get [here](https://en.wikipedia.org/wiki/HyperLogLog "Wikipedia:HyperLogLog").
The main reason to use HLL is necessity to estimate uniques in very big amount of data in case if it is possible to sacrifice accuracy of an unique counter.

## List of implementations

You can find several implementations of HLL:

* [twitter/algebird](https://github.com/twitter/algebird "twitter/algebird") - a scala library from twitter which contains lots of different algorithms including HLL
* [prasanthj/hyperloglog](https://github.com/prasanthj/hyperloglog "prasanthj/hyperloglog") - a detached java library for HLL 
* [addthis/stream-lib](https://github.com/addthis/stream-lib "addthis/stream-lib") - another java lib which have an implementation of HLL.

Next in this article we will take a close look at all these libs and answer the question: "Why should we use HLL?".

<!--more-->

## Example of use
So why should we might use HLL in our programs? The main reason is reducing of data which we have to store.

For example you need to count the number of unique visitors for every page on a site. You can use a structure like this:

```scala
case class PageVisitors(pageUrl: String, visitors: Set[UUID])
```

And it will work until you have a lot of pages and very many visitors on each page.
For instance let`s assume that we have an analytic system which work with visit statistic of different sites. Suppose we have 1000 pretty popular sites and because sites are popular each site have about 100000 unique visitors per day. 
We also want to save statistic of unique visitors per day for each site at least one year. We mark visitors using UUID - 128 bits identifier.

```
one UUID = 128 bits = 16 bytes per unique visitor

# for one site per day
100000 unique visitors * 16 bytes = 1600000 bytes ~= 1.52 Mbyte of data 

# for one site per year
1.52 Mbyte * 365 days = 556 Mbyte 

# total amount of data for 1000 sites
556 Mbyte * 1000 = 556 Gbyte of total data
```

But if we are ready to sacrifice accuracy a little bit we will considerably reduce amount of stored data.
For example if we use `prasanthj/hyperloglog` to store unique counter `16Kbytes` will be enough to store `10.000.000` unique values only with `~0.52%` of error.

```
16Kbyte per site * 365 days * 1000 sites = 5.5Gbyte of total data
```

We will reduce the needed size of data a hundredfold if we agree with 0.52% of error.

## twitter/algebird

This library contains various algorithms like `HyperLogLog`, `CountMinSketch` and others. The full list you can find [here](https://github.com/twitter/algebird/wiki "Algebird docs").

Below is an example of using `algebird.HyperLogLogMonoid`:

```scala
object SimpleAlgebirdExample {
  import com.twitter.algebird.HyperLogLogMonoid

  def main(args: Array[String]) {
    //define test data
    val data = Seq("aaa", "bbb", "ccc")
    //create algebird HLL
    val hll = new HyperLogLogMonoid(bits = 10)
    //convert data elements to a seq of hlls
    val hlls = data.map { str =>
      val bytes = str.getBytes("utf-8")
      hll.create(bytes)
    }

    //merge seq of hlls in one hll object
    val merged = hll.sum(hlls)

    //WARN: don`t use merged.size - it is a different thing
    //get the estimate count from merged hll
    println("estimate count: " + hll.sizeOf(merged).estimate)
    //or
    println("estimate count: " + merged.approximateSize.estimate)
  }
}
```

## prasanthj/hyperloglog
It is a java library containing only an implementation of `HyperLogLog`. 
There is no a maven artefact for this library, so you can build it manually or use [jitpack.io](https://jitpack.io/ "jitpack.io") as it was described at the end of [this article](http://koff.io/posts/291279-true-fail-fast-with-expression-magic).

Also it contains a command line test tool which can help to choose settings for HLL and to see how accurate estimation will be. In order to use it you should build  the library locally using `maven` and execute `hll` script.

```bash
# clone repo
git clone https://github.com/prasanthj/hyperloglog.git hyperloglog
cd hyperloglog

# build the library
mvn package -DskipTests

# execute tests
./hll -n 10000000 -s -o ./out.hll
```

You should see something like that:

```
Actual count: 10000000
Encoding: DENSE, p : 14, chosenHashBits: 128, estimatedCardinality: 10052011
Relative error: -0.52011013%
Serialized hyperloglog to ./out.hll
Serialized size: 10248 bytes
Serialization time: 13 ms
```

You can find out more details about params of `hll` on the [github page](https://github.com/prasanthj/hyperloglog "prasanthj/hyperloglog") of the library.

The example of how to use it is below:

```scala
object SimplePrasanthjHllExample {
  import hyperloglog.HyperLogLog.HyperLogLogBuilder
  def main(args: Array[String]) {
    //define test data
    val data = Seq("aaa", "bbb", "ccc")

    //create a builder for HLL.
    val hllBuilder = new HyperLogLogBuilder()
    // You can set different parameters for it using
    // hllBuilder.setEncoding(...)
    // hllBuilder.setNumHashBits(...)
    // hllBuilder.setNumRegisterIndexBits(...)
    // hllBuilder.enableBitPacking(...)
    // hllBuilder.enableNoBias(...)

    //create hll object in which we will merge our data
    val mergedHll = hllBuilder.build()

    //merge data
    data.foreach { elem =>
      //explicitly set using encoding
      val bytes = elem.getBytes("utf-8")
      mergedHll.addBytes(bytes)
    }

    //print the estimation of count
    println("estimate count: " + mergedHll.count())
  }
}
```

## addthis/stream-lib
It is another java library where you can find lots of algorithms. But I have not found any documentation for it. So it can be a problem. If you have a question how to use this library you can try to ask it in the mailing list [here](http://groups.google.com/group/stream-lib-user)

The simple example is here:

```scala
object SimpleStreamExample {
  import com.clearspring.analytics.stream.cardinality.HyperLogLogPlus
  def main(args: Array[String]) {
    //define test data
    val data = Seq("aaa", "bbb", "ccc")

    //create HLL object in which we will add our data.
    // You can set parameters here in a constructor
    val merged = new HyperLogLogPlus(5, 25)

    //adding data in hll
    data.foreach{ elem =>
      //in order to control string encoding during string conversion to bytes we explicitly set using encoding
      val bytes = elem.getBytes("utf-8")
      merged.offer(bytes)
    }

    //print the estimation
    println("estimate count: " + merged.cardinality())
  }
}
```

## Bonus: Intersection of HLLs
It is possible to make intersections of HLL counters in order to find the number of common elements in them. Such logic is implemented in `twitter/algebird` [here](https://github.com/twitter/algebird/blob/develop/algebird-core/src/main/scala/com/twitter/algebird/HyperLogLog.scala#L601).
The main underlying principle in this algorithm is ["Inclusion–exclusion principle"](https://en.wikipedia.org/wiki/Inclusion%E2%80%93exclusion_principle). 
But you should be aware about how intersection affect an estimation error. I have found [this article](http://research.neustar.biz/2012/12/17/hll-intersections-2/ "HLL Intersections") which helps to understand possible issues.

In order to make possible computation of intersection for other libs I have also created small facade which can be found in the package `io.koff.hll.facade`. 
And below you can see the example of how to use it in order to find intersection of four hlls:

```scala
package io.koff.hll

/**
 * A common algorithm to find intersection of several HLL counters
 */
object Intersection {
  import io.koff.hll.facade._
  import io.koff.hll.facade.impl.algebird._
  def main(args: Array[String]) {
    //"444", "555" and "666" are common elements
    def set1 = Seq("111", "222", "333", "444", "555", "666").toHLL
    def set2 = Seq("222", "333", "444", "555", "666", "777").toHLL
    def set3 = Seq("333", "444", "555", "666", "777", "888").toHLL
    def set4 = Seq("444", "555", "666", "777", "888", "999").toHLL

    val intersectionCount = HLLUtils.intersection(set1, set2, set3, set4)
    println("intersection: " + intersectionCount) //will print "intersection: 3"
  }
}

```

## Comparison Results
I have also written a little test which compares described libraries. You can find it [here](https://github.com/coffius/koffio-hll/blob/master/src/main/scala/io/koff/hll/Comparison.scala "GitHub: source code").
Results you can see below:

```
algebird	[error: 0,007217%, calcTime: 8445 msecs, estimateCount: 992784, dataSize: 65536 bytes]
prasanthj	[error: 0,003138%, calcTime: 559  msecs, estimateCount: 996863, dataSize: 10247 bytes]
stream		[error: 0,002794%, calcTime: 293  msecs, estimateCount: 997207, dataSize: 43702 bytes]
```

So as you can see:

* `twitter/algebird` has showed itself as quite a slow library. It can be a problem if you work with big data.
* `prasanthj/hyperloglog` always needs minimal space to store serialized data of HLL counter.
* `addthis/stream-lib` is the fastest library in those that we have looked at.

## Links

* [Example sources](https://github.com/coffius/koffio-hll "GitHub: coffius/koffio-hll") - an example project can be found here
* [twitter/algebird](https://github.com/twitter/algebird "twitter/algebird") - a scala library from twitter which contains lots of different algorithms including HLL
* [prasanthj/hyperloglog](https://github.com/prasanthj/hyperloglog "prasanthj/hyperloglog") - a detached java library for HLL 
* [addthis/stream-lib](https://github.com/addthis/stream-lib "addthis/stream-lib") - another java lib which have an implementation of HLL.
* [HLL Intersections](http://research.neustar.biz/2012/12/17/hll-intersections-2/) - an article about accuracy of the HLL`s intersections.