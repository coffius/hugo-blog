+++
date = "2015-10-13T22:01:13+03:00"
draft = false
title = "Using T-Digest: Median calculation and anomaly detection"
slug = "using-t-digest"
+++

In this article you can find information about using `t-digest` library in order to measure average value of some quantity(average session time). 
There is also an answer for the question: What and why should you use to make such the measurement mean or median? 
Besides, list and comparison of different implementations is presented below in the article.

## Main problem
So in what cases we have need to calculate mean/median? For example we have a site and we want to understand how much time an average user spent on our site. 
In order to do it we should calculate an average duration of a user session.
And there are at least two ways to do it - calculate an arithmetic mean(or just mean) or calculate a median.
The calculation of mean is very simple. You need two fields: one for a sum of elements and another for their count. But it doesn`t work very well with anomalies in the data. 
I mean the case when one or several elements differ greatly from others.
Lets assume that we have such values for our session durations(in milliseconds):

```
3000, 2000, 3000, 5000, 3000, 4000, 4500, 3200, 2700, 3380
```
mean = `(3000+2000+3000+5000+3000+4000+4500+3200+2700+3380) / 10 = 3378 msecs`. In this case all is ok.

But what if one of these users opens the site, forgets to close a browser tab and goes afk for an hour(3.600.000 msecs):

```
3000, 2000, 3000, 5000, 3000, 4000, 4500, 3200, 2700, 3600000
```

mean = `(3000+2000+3000+5000+3000+4000+4500+3200+2700+3600000) / 10 = 363040 msecs`. Just one of the users influences strongly on mean value. 
Generally speaking, the mean is only representative if the distribution of the data is symmetric, otherwise it may be heavily influenced by outlying measurements.
In simple cases it is possible to use some kind of a filter.
But often we just don't know what threshold we should use to filter values. 
Whereas the median value is the same in both cases and is equal to `3100`. So in the cases like this the median will be more useful then the mean.
However the calculation of the median in general case needs a lot of memory - O(n)

<!--more-->

## T-digest

In order to deal with memory consumption [`Q-Digest` algorithm](http://www.cs.virginia.edu/~son/cs851/papers/ucsb.sensys04.pdf "Pdf: Medians and Beyond: New Aggregation Techniques
for Sensor Networks") has been developed and after that `T-digest` has been created as improvement of `Q-digest`. 
The main idea of these algorithms is to sacrifice accuracy of calculation for decrease of required amount of memory.

## List of implementations

I've found these libs for median calculation:

* [tdunning/t-digest](https://github.com/tdunning/t-digest "GitHub: t-digest") - a java library with implementation of `t-digest` algorithm
* [addthis/stream-lib](https://github.com/addthis/stream-lib "GitHub: stream-lib") - we already saw it in [previous article](/posts/comparison-of-hll). It's a library of streaming algorithms written in Java.
* [apache/commons-math](http://commons.apache.org/proper/commons-math/ "Commons Math: The Apache Commons Mathematics Library") - it's a library of different mathematics and statistics components from apache.

`apache/commons-math` implements a regular algorithm for median calculation. It is used for the estimation of calculation accuracy of viewed `t-digest` implementations.

## tdunning/t-digest: simple example
A short description from github page:

```text
A new data structure for accurate on-line accumulation of rank-based statistics such as quantiles and trimmed means. The t-digest algorithm is also very parallel friendly making it useful in map-reduce and parallel streaming applications.
```

Friendliness for a parallel calculation shows itself by support of a merging operation. You can see it in the example below.

Example of the code for t-digest:

```scala
object TDigestExample {
  import com.madhukaraphatak.sizeof.SizeEstimator
  import com.tdunning.math.stats.TDigest

  def main(args: Array[String]) {
    //define constants for experiments
    val oneSecond = 1000
    val twoMinutes = 2 * 60 * 1000
    val tenMinutes = 10 * 60 * 1000
    val twoHours = 2 * 60 * 60 * 1000

    val mainValues = 10000000
    val badValues = 100000

    //generate 10.000.000 pseudo-random values for normal user session durations
    val mainData = Generator.generate(count = mainValues, from = oneSecond, to = twoMinutes)

    //generate 100.000(1%) pseudo-random values for invalid user session durations
    val badData = Generator.generate(count = badValues, from = tenMinutes, to = twoHours)

    //generate one united collection. For further details see below
    val totalData = {
      Generator.generate(count = mainValues, from = oneSecond, to = twoMinutes) ++
        Generator.generate(count = badValues, from = tenMinutes, to = twoHours)
    }

    // Experiment #1
    // All data in one digest object
    // all values from one collection(totalData) are added to one digest object

    // recommend default value of compression = 100
    val totalDigest = TDigest.createArrayDigest(100)

    //the timing of the median calculation
    val startTime = System.currentTimeMillis()
    totalData.foreach(value => totalDigest.add(value))
    val median = totalDigest.quantile(0.5d)               //the median is a 0.5 quantile
    val calcTime = System.currentTimeMillis() - startTime
    println("calcTime: " + calcTime)

    // Experiment #2
    // Separate objects for normal(mainData) and bad data(badData) to check accuracy of t-digest merging
    val mainDigest = TDigest.createArrayDigest(100)
    mainData.foreach(value => mainDigest.add(value))

    val badDigest = TDigest.createArrayDigest(100)
    badData.foreach(value => badDigest.add(value))

    //to check accuracy of merging several digests
    val mergedDigest = TDigest.createArrayDigest(100)
    mergedDigest.add(mainDigest)
    mergedDigest.add(badDigest)

    val mergedMedian = mergedDigest.quantile(0.5d)
    println("median:       " + median)
    println("mergedMedian: " + mergedMedian)

    //how many bytes is needed to serialize t-digest object
    println("byte size:    " + totalDigest.byteSize())

    //how much memory is needed for totalDigest object
    val sizeInMem = SizeEstimator.estimate(totalDigest)
    println(s"size in mem:  $sizeInMem bytes")
  }
}
```

## addthis/stream-lib

In general there is no much difference in the interface between `tdunning/t-digest` and `addthis/stream-lib`. The example for `addthis/stream-lib` you can find in [`StreamLibExample.scala` file](https://github.com/coffius/koffio-t-digest/blob/master/src/main/scala/io/koff/t_digest/StreamLibExample.scala). 
However `tdunning/t-digest` has more accuracy and less calculation time. More details about lib comparison are below.

## Comparison
Now lets compare our libraries. The libraries are compared by:

* Time of calculation
* Calculation accuracy
* Serialization size
* Needed memory

**Results:**

|                  | Median Value | Calc.Time(msec) | Calc. Acc.(%) | Serial. size(bytes) | Memory(bytes) |
|------------------|--------------|-----------------|---------------|---------------------|--------------:|
| **commons-math** |   61084,0000 |           5.922 |    100,000000 | not-supported       |   134.218.824 |
| **t-digest**     |   61084,6668 |           8.234 |    100,001090 | 15740               |        29.688 |
| **stream-lib**   |   60763,2110 |          20.142 |     99,474839 | 24820               |       231.784 |

## Anomaly detection

One more interesting application of the median is anomaly detection. In order to find anomalies in your data sequence threshold should be calculated. 
For this you need to get 99.9%-quantile which means that we expect ~0.1% of anomalies in the data sequence. Look below to see how to do it using `tdunning/t-digest`.

```scala
/**
 * Example of anomaly detection
 */
object AnomalyDetectionExample {
  import com.tdunning.math.stats.TDigest
  def main(args: Array[String]) {
    //define constants for experiments
    val oneSecond = 1000
    val twoMinutes = 2 * 60 * 1000
    val tenMinutes = 10 * 60 * 1000
    val twoHours = 2 * 60 * 60 * 1000

    val mainValues = 10000000
    val badValues = 10000

    //generate 10.000.000 pseudo-random values for normal user session durations
    val mainData = Generator.generate(count = mainValues, from = oneSecond, to = twoMinutes)

    //generate 100.000(1%) pseudo-random values for invalid user session durations
    val badData = Generator.generate(count = badValues, from = tenMinutes, to = twoHours)

    val totalData = mainData ++ badData

    val totalDigest = TDigest.createArrayDigest(100)

    totalData.foreach(value => totalDigest.add(value))
    //this threshold means that we expect ~0.1% of data is anomalies
    val threshold = totalDigest.quantile(0.999d).toInt

    println(s"threshold: $threshold msec")
  }
}
```

## Links

* [Example project](https://github.com/coffius/koffio-t-digest "GitHub: Example project") - sources for this article
* [tdunning/t-digest](https://github.com/tdunning/t-digest "GitHub: t-digest") - github page of `t-digest`
* [addthis/stream-lib](https://github.com/addthis/stream-lib "GitHub: stream-lib")
* [apache/math](https://commons.apache.org/proper/commons-math/) - the apache commons mathematics library