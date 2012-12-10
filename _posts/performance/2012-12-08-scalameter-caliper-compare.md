---
title: Comparison of Scalameter and Caliper
layout: post
category : peformance
tags : [scalameter, caliper, benchmark, scala]
---

# Benchmark results

### Timing a while loop

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

### The results

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

### Timing finding the 40th fibonacci number

The code:

{% highlight scala %}
def fibonacci(n:Int):Unit = {
  if (n < 0) throw new IllegalArgumentException("n = " + n + " < 0")
  def _f(x:Int):Int = {
    if (x <= 1) n else { _f(x - 1) + _f(x - 2) }
  }
  _f(n)
}                          
{% endhighlight %}

### The results
Caliper:
{% highlight bash %}
[info] 80% Scenario{vm=java, trial=0, benchmark=fibonacci} 556474178.00 ns; σ=4617459.68 ns @ 3 trials
[info] 
[info]            benchmark     ms linear runtime
[info]            fibonacci 556.47 
{% endhighlight %}


Scalameter:
{% highlight bash %}
[info] ::Benchmark fibonacci::
[info] cores: 4
[info] hostname: Quasar
[info] jvm-name: Java HotSpot(TM) Server VM
[info] jvm-vendor: Oracle Corporation
[info] jvm-version: 21.0-b17
[info] os-arch: i386
[info] os-name: Linux
[info] Parameters(limit -> 40): 541.119767
{% endhighlight %}




I noticed with consecutive runs that the run times reported by scalameter were consistently shorter than those reported by caliper.


# How long do they take to run?

# Why the difference?


I want to create a side by side micro comparison between running various benchmarks with caliper and scalameter side by side. I want to encourage the community to run on a wide range of computers, at which point it will upload the results to an endpoint I created. This will hopefully produce a large dataset that I can write a second part post about. Basically a performance data set collection that compares the results of two different methods of running performance timings against a range of sbt-enabled environments.

Peeling back caliper, to use the execution. Peeling back scalameter, keeping the interface and reporting. Moving caliper's console reporting to scalameter reporting, also any of the html functionality that I have yet to work with. Solder the peices at the points of execution and measurement results.

I will have to combine the best peices of test configuration. Might as well take all the good features. This seems to be an area that both libraries could work on, scalameter in the setting of more global options, caliper with general ease of use and defaults. 

I'm trying to keep an even level of understanding of different parts of the code, so that I can keep in mind where the solder points appear. I need to peel back caliper enough to find the benefits of using it's internals, if there are in fact reasons to do so.

Considering the code where caliper creates the actual process that runs the benchmark code. It creates a process and records measurements through that process's input stream. How does scalameter interact with the benchmark jvm subprocess?

Scalameter runs the benchmark in a seperate process using the following command:
{% highlight scala %}
val command = s"java -server $flags -cp $classpath ${classOf[Main].getName} ${tmpfile.getPath}"
{% endhighlight %}
Nice to see some scala 2.10 string interpolation.

Caliper runs the benchmark jvm subprocess using settings from the commandline and vm selection options.
Caliper reads results through stdin as a json object.

Imagining running performance benchmarks across a small cluster of varied architectures. Reporting mechanism for suspicious results. The implications on scaling that system up would be a distrbuted way to measure runtimes of server responses, and report geolocation of suspicious activity, upon which time the cluster can focus around that area to verify results. If verified, alerts created. (Idea more for network architecture). 

- Running int Array while loop benchmark:

Scalameter is consistently (+/- less than a millisecond) reporting around 37.5 ms. Caliper is reporting 4.70, 4.70, 4.66, 4.68, 4.60. This is a very big difference. 

{% highlight bash %}
Everthing is created from a process. The process is directly reflected in the result. What is your process? How do you learn your process, hone your process?
{% endhighlight %}

To investigate, I need to have control over running both scalameter and caliper at the point which they spawn the process. This is the point where both call a runner class (in caliper InProcessesRunner, in scalameter Main) that runs a specific benchmark test, and encodes the results in object format that is sent to the parent process. In caliper, the measurement encoding data type is MeasurementSet, which it transports between processes as a json object through stdin once the test is finished. Scalameter encodes its measurements in the data type into Seq[(Parameters, Seq[Double])]. It uses a temp file to communicate the serialized java code that defines the method to run the benchmarks, and reads in the same file as a serialized version of the data. 

Notes on possible performance regression execution path: Tests are run and compared to some mean value. If test result is outside of threshold, test is run again to try and detect an actual anomoly. If test is deemed of different performance, a profiler is run against the code. That profile is compared to a historical profile, to try and account for the difference in performance results. It will report the variance in performance as well as an analysis of what functions\call stacks caused the difference. This is what scalpel will be. It will not just be for regression. It will be for performance tuning. You can change code paths, and see how that changes profiles and overall timing results.

One advantange of caliper's interprocess json object communication is that it would transform nicely to calling performance measurements as REST services.

To run in consecutive slices:
  For scalameter, to get curve data, call SeperateJvmsExecutor.runSetup.
  For caliper, to get a MeasurementResult, call measure in CaliperRunner.
  
Now scalameter is reporting the benchmark only takes around 5.2 ms. Consistently.
Caliper is still reporting 4.60,4.71...


*The code for the comparison can be found [here](http://github.com/lossyrob/scalameter-caliper)*



