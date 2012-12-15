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
Now scalameter is back up to 38.08!

## Monday, December 10

Running scalameter and caliper back to back now. Here's the results of running a the int while loop in each, consecutively:


    [info] Running scalpel.ScalpelRunner 

    Caliper measure time: 4.615950835680751 ms
    Scalameter time data: 38.170374

    Caliper measure time: 4.386483468208092 ms
    Scalameter time data: 38.390352

    Caliper measure time: 4.599671330275229 ms
    Scalameter time data: 37.580277

    Caliper measure time: 4.601634201834862 ms
    Scalameter time data: 37.494863

    Caliper measure time: 4.6161024272445825 ms
    Scalameter time data: 38.304218

    Caliper measure time: 5.747781575144509 ms
    Scalameter time data: 5.826549
    [success] Total time: 180 s, completed Dec 11, 2012 12:10:28 AM

As you can see, the last run through is the only one where there is matching, and caliper reports a 1 ms performance hit. What is going on here?

This is the result of running caliper and scalameter again, but with the first scalameter run being with a SeperateJvmsExecutor and the second with a LocalExecutor:

    [info] Running scalpel.ScalpelRunner 

    Caliper measure time: 4.5892122018348624 ms
    Scalameter time data: 37.545266
    Scalameter time data: 4.313085

    Caliper measure time: 4.642768888888889 ms
    Scalameter time data: 38.154935
    Scalameter time data: 4.216995

    Caliper measure time: 4.665680329192546 ms
    Scalameter time data: 37.629374
    Scalameter time data: 4.204

    Caliper measure time: 4.627782194444444 ms
    Scalameter time data: 38.260855
    Scalameter time data: 4.344657

    Caliper measure time: 4.643496628273662 ms
    Scalameter time data: 38.834667
    Scalameter time data: 4.318518

    Caliper measure time: 4.67964090625 ms
    Scalameter time data: 38.338802
    Scalameter time data: 4.302616

Clearly something is going awry with scalameter's SeperateJvmsExecutor. The local executor seems to be consistent, and line up at least down to the millisecond with caliper. So the question is, what is wrong with scalameter's SeperateJvmsExecutor logic? Why does it tack on an extra 34 milliseconds to the measurments, consistently but not constantly?

The next step is to really understand the internals of caliper, and know:
    
* how the benchmark subprocess executes the measurements
* What preperations are done to the JVM in order to prepare it for the benchmark code

The scalameter subprocess benchmark...

## Google Caliper InProcessRunner

The InProcessRunner class is what is called to run on the benchmark subprocess jvm. It is called with the suite class name. 

- The InProcessesRunner parses the arguments to extract the proper Scenario and Measurer. 
- The measurer has a run function which is called with a Supplier[ConfiguredBenchmark]. Some oldie but goodie factory design patterns. All to create () => scenarioSelection.CreateBenchmark(scenario). A great example of the tersness of having functions as first class citizens **say better**. There's AllocationMeasurer, DebugMeasurer, and TimeMeasurer. TimeMeasurer taks as parameters warmup milliseconds and run millisceconds. The warmup time is how long a while loop executes that runs the benchmark, as a warmup run.

Scalameter's final run code:


    val start = System.nanoTime
    snippet(value)
    val end = System.nanoTime

Caliper's final run code:

    long startNanos = System.nanoTime();
    benchmark.run(reps);
    long endNanos = System.nanoTime();
 
Let's explore what the lines benchmark.run(reps) and snippet(value) are really doing.

### benchmark.run(reps)
benchmark is of type ConfiguredBenchmark. ConfiguredBenchmark is an abstract class with a run. Except for run and close, which are abstract, it is a simple wrapper of a Benchmark. 
Simplebenchmark createsBenchmark with parameter values, including the function name. A ConfiguredBenchmark is returned with the run function calling the benchmark method with the 
number of reps. So, run is a function defined on an anonymous instance of ConfiguredBenchmark, that is assigned to run the method *on a copy of the benchmark using reflection*. My assumption
is that this more direction link to the method might be the answer to the difference in execution times.

### snippet(value)
snippit is an anonymous function passed in as a function argument that is of type T => Any. Right there it seems like a direct possibility for slowness with boxing parameter values. Caliper calls a function with one integer parameter in a non-boxed way.

The 'Executor' in scalameter is the solder point. Specifically, the code that runs the new JVM (which seems very silimar between caliper and scalameter, no big difference there) should be the InProcessRunner of caliper. The in processer runner should execute how caliper executes. It should be modified to return Scalameter's measurement data type. At that point scalameter's reporting framework would display the data. This reporting framework should encorporate the ConsoleReporter as the default log reporter (display the caliper histogram, because it's awesome and as a point of reference to initial users). 

The current path to port caliper's internals into scalpel will be to continue to peel back caliper, rewriting the peeled code to use CurveData in scala.
First I need to have caliper run through scala meter. Then pinch out the point of translation between scalameter configuration internal type representaitons into caliper external type representations, connecting the internal representaitons of measurement data.

To be able to create my own versions of data types, translation objects will contain functions that take one project's data type and parse it into an internal data type. There will be getters that will translate it the data typed used by the second project. In this way the 'translator' objects are mapping functions TScalaMeter => TInternal and TInternal => TCaliper in the case of benchmark setup, or TCaliper => TInternal and TInternal => TScalaMeter in the case of measurement data (MeasurementSet -> CurveData). 

Create translation methods described above for solder points, then push down java code and forward scala code for port.

## Wednesday, December 12 - 0058e40 

Fuck yeah! I have caliper almost executing tests dictated by scalameter. I've got a game plan. We're in business.

The next step will be to flesh out the translation layer. I've been trying pretty hard to keep the scalpel namespace clean, and only reference scalameter and caliper type with qualified names. Also, I've created a 'port' namespace that I will use to transition caliper code from java to scala. Anything with 'port' is in caliper land.

After fleshing out the translation layer, which should end in full test running from scalameter of caliper benchmarks, I want to try and fix the problem of abstraction between the benchmark types.

## Definition of Benchmarks

Caliper defines it's benchmark as a class containing a set of methods, taking only an integer parameter, that are to be **directly called** from the benchmark running code (InProcessRunner). Scalameter defines a very usable and intuitive DSL to define benchmarks. It walks up a tree of definitions and creates a Setup[T] that represents the benchmark. Included in that setup is an abstraction of the benchmark code. When the function that is passed to the seperate JVM for execution is serialized, it closes on all the state necessary for executing the function, including the setup, containing the benchmark code. This is a lot of abstraction between what actually calls the benchmark code, and how it ends up running it. With caliper, a copy of the benchmark object is made (so all state is held in the JVM in an object), and the method is called directly on that object representation. The less you have between the call of the benchmark code and the code under test, the more accurate the measurement. All code between the function call in between System.nanoTime calls and the code under test is a liability. 

* **The goal is to reduce that liability by making the call-to-code footprint as small as possible **

I'm hoping macros could help in allowing as beautiful of an interface as scalameter has, while beating both of them in results.

Tonight's goal: port caliper.MeasurementSet.

It's not going to be that easy, class based....it's probably going to happen in big steps. Finding the cutoff points at each step is going to be tricky. But I must continually be leaping to compile points, and that requires scope.

[check irc logs]

## Saturday, December 15th

[irc]
The intrusion technique did not work. It caused deep problems that caused caliper to stop running. Even with only supplanting Measurement. Learning from mistakes.

Plan of attack: Create ScalpelBenchmark, which uses CaliperExecutor, get it to run. That is the solder point. Then push through the port of the main internals from there. There will have to be two classes, a seperate benchmark class to contain the caliper code, one SMBenchmark that contains the SM code. Then port the ConsoleReport because it's awesome and I want the output.

# Code
The code for the comparison can be found [here](http://github.com/lossyrob/scalameter-caliper) and [here](http://github.com/lossyrob/scalpel)

