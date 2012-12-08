---
title: Comparison of Scalameter and Caliper
layout: post
category : peformance
tags : [scalameter, caliper, benchmark, scala]
---

# Timing a while loop

First, let's set up some arrays to use in out benchmarks:
{% highlight scala %}
def init[A:Manifest](size:Int)(init: => A) = {
  val data = Array.ofDim[A](size)
  for (i <- 0 until size) data(i) = init
    data
  }

  val len = 1000000
  ints = init(len)(Random.nextInt)
  doubles = init(len)(Random.nextDouble)
{% endhighlight %}

Here is the code we will use for our benchmarks:
{% highlight scala %}
    // Code to be tested for integers
    val goal = ints.clone
    var i = 0
    val len = goal.length
    while (i < len) {
      val z = goal(i)
      if (z != Int.MinValue) goal(i) = z * 2
      i += 1
    }
{% endhighlight %}

{% highlight scala %}
    // Code to be tested for doubles
    val goal = ints.clone
    var i = 0
    val len = goal.length
    while (i < len) {
      val z = goal(i)
      if (z != Int.MinValue) goal(i) = z * 2
      i += 1
    }
{% endhighlight %}

## The results

Caliper:
{% highlight bash %}
> run
[info]  0% Scenario{vm=java, trial=0, benchmark=IntArrayWhileLoop} 4337952.77 ns; σ=276850.32 ns @ 10 trials
[info] 25% Scenario{vm=java, trial=0, benchmark=DoubleArrayWhileLoop} 62181398.67 ns; σ=305417.34 ns @ 3 trials
[info]
[info]            benchmark    ms linear runtime
[info]    IntArrayWhileLoop  4.34 ==
[info] DoubleArrayWhileLoop 62.18 ==============================
[info]
[info] vm: java
[info] trial: 0

{% endhighlight %}

Scalameter:
{% highlight bash %}
[info] ::Benchmark while loop.int array::
[info] cores: 4
[info] hostname: Quasar
[info] jvm-name: Java HotSpot(TM) Server VM
[info] jvm-vendor: Oracle Corporation
[info] jvm-version: 21.0-b17
[info] os-arch: i386
[info] os-name: Linux
[info] Parameters(ints -> [I@14294b0): 3.95428
[info]
[info] ::Benchmark while loop.double array::
[info] cores: 4
[info] hostname: Quasar
[info] jvm-name: Java HotSpot(TM) Server VM
[info] jvm-vendor: Oracle Corporation
[info] jvm-version: 21.0-b17
[info] os-arch: i386
[info] os-name: Linux
[info] Parameters(doubles -> [D@160b783): 61.484498

{% endhighlight %}

I noticed with consecutive runs that the run times reported by scalameter were consistently shorter than those reported by caliper.

*The code for the comparison can be found [here](http://github.com/lossyrob/scalameter-caliper)*



