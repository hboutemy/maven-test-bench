<div align="center">
<blockquote>
<p><h3>This project is a test bench to measure and investigate heap allocation of Apache Maven.</h3></p>
</blockquote>
</div>

<p align="center">
  <a href="#General-setup">General setup</a> •
  <a href="#Benchmark-heap-allocation-of-several-Maven-releases">Benchmark heap allocation of several Maven releases</a>
</p>
<p align="center">
  <a href="#Investigate-where-heap-allocation-comes-from">Investigate where heap allocation comes from</a> •
  <a href="#Acknowledgments">Acknowledgments</a> •
  <a href="#License">License</a> 
</p>

At this moment, this project allows to benchmark and investigate the origin of heap allocation caused by `mvn validate` (the first phase before lanching any plugin: see [Lifecycles Reference](https://maven.apache.org/ref/current/maven-core/lifecycles.html)).

This project uses [QuickPerf](https://github.com/quick-perf/quickperf) to measure and investigate the heap allocation level.

Measures have been done executing `mvn validate` on Apache Camel source code. 

Feel free to use this project and contribute to it!

# General setup

This project contains two types of test:
- `MvnValidateAllocationByMaven3VersionTest` can be used to evaluate the heap allocation level for a range of Maven versions,
- for a given Maven version, `MvnValidateProfilingTest` can be used to investigate the origin of allocation.

This general setup part describes configurations common to both tests.

You have to define values to the `project-under-test.path` and `maven.binaries.path` properties contained in the `maven-bench.properties` file. The other properties are only used by `MvnValidateAllocationByMaven3VersionTest`.

The `project-under-test.path` represents the path of the project on which `mvn validate` will be applied. 
Our next measures are based on the Apache Camel project, but you can choose your own target. For reproducibility of our measure, a precisely defined version of this project was chosen:
```
git clone -n https://github.com/apache/camel.git
git checkout c409ab7aabb971065fc8384a861904d2a2819be5
```
This Apache Camel version contains 841 modules: such a huge build is perfect to get significant measures.

The `maven.binaries.path` property corresponds to the path where the needed Maven distributions will be automatically downloaded by the tests. Downloads are performed during *@Before* execution.
If you want to apply measures on Maven HEAD, you can execute the following commands where {maven-distrib-location} has to be replaced with the url given by the `maven.binaries.path` property of `maven-bench.properties` file:
```
git clone https://github.com/apache/maven.git
cd maven
mvn -DdistributionTargetDir="{maven-distrib-location}/apache-maven-head" clean package
``` 

# Benchmark heap allocation of several Maven releases

`MvnValidateAllocationByMaven3VersionTest` test allows to benchmark the heap allocation level on several Maven 3 distributions.

Heap allocation level is measured with the help of [@MeasureHeapAllocation](https://github.com/quick-perf/doc/wiki/JVM-annotations#Verify-heap-allocation) QuickPerf annotation. This annotation measures the heap allocation level of the thread running the method annotated with @Test.
Feel free to contribute to QuickPerf by adding a feature allowing to measure the allocation level aggregated across all the threads! With `mvn validate`, we have checked that Maven code is not multithreaded during this validate phase by profiling the JVM with the help of [@ProfileJvm](https://github.com/quick-perf/doc/wiki/JVM-annotations#ProfileJvm).

Please read [General setup](#General-setup) to get some of the setup requirements.

You also have to give a value for the following properties contained in the [maven-bench.properties](src/test/resources/maven-bench.properties) file:
* `maven.version.from`
* `maven.version.to`
* `warmup.number`
* `measures.number-by-maven-version`

The meaning of these properties is given in the [maven-bench.properties](src/test/resources/maven-bench.properties) file.

Measures can be launched with this command line: ```mvn -Dtest=MvnValidateAllocationByMaven3VersionTest test```.
Before doing it, you can close your IDE, web browser or other applications to free memory.

The benchmark results are exported into a `maven-memory-allocation-{date-time}.csv` file. The execution context (processor, OS, ...) is reported in an `execution-context-{date-time}.txt` file.

For several Maven versions, the following graph gives the average of ten heap allocations caused by the application of `mvn validate` on Apache Camel:
<p align="center">
    <img src="measures/mvn-validate-on-camel.png">
</p>

For this graph, you can consult:
* [the measures](measures/maven-memory-allocation-2019-09-01-18-48-41.csv)
* [the execution context](measures/execution-context-2019-09-01-18-48-41.txt)

Measures took 1 hour and 12 minutes. From Maven versions 3.2.5 to 3.6.2, heap allocation level is the highest with Maven 3.2.5 and the smallest with Maven 3.6.2. And __the heap allocation decreases from ~7 GB with Maven 3.6.1 to ~3 GB with Maven 3.6.2__.

JVM heap size can be fixed with the help of [@HeapSize](https://github.com/quick-perf/doc/wiki/JVM-annotations#heapsize):
* with Maven 3.2.5 and a JVM heap size between 6 and 9 GB, one measure of heap allocation takes around one minute: the test duration is about 1.5 minute with a 5 GB heap size, probably due to more garbage collection,
* __with Maven 3.6.2, the test duration drops to 15 seconds for a heap size between 1 and 9 GB__

Less heap allocation means you can allocate less memory, and given a memory allocation you get less garbage collection, then it takes less time: with QuickPerf, we were able to measure the improvement precisely.

# Investigate where heap allocation comes from

You can use `MvnValidateProfilingTest` to understand the origin of heap allocation.
Some of the set up requirements can be found in [General setup](#General-setup) part.

The Maven version under test can be set with the `MAVEN_3_VERSION` constant:
``` java
    public static Maven3Version MAVEN_3_VERSION = Maven3Version.V_3_6_2;
```

A test method is annotated with [@ProfileJvm](https://github.com/quick-perf/doc/wiki/JVM-annotations#Profile-or-check-your-JVM) to profile the test method with Java Flight Recorder (JFR).

The JFR file location is going to be displayed in the console:
```
[QUICK PERF] JVM was profiled with Java File Recorder (JFR).
The recording file can be found here: C:\Users\JEANBI~1\AppData\Local\Temp\QuickPerf-46868616\jvm-profiling.jfr
You can open it with Java Mission Control (JMC).
```

You can open it with Java Mission Control (JMC). 

Below a JFR file for Maven 3.2.5 and opened with JMC 5.5:
<p align="center">
    <img src="measures/Maven3.2.5-JMC.5.5JPG.jpg">
</p>


By the way, you can also benefit from an automatic performance analysis with [@ExpectNoJvmIssue](https://github.com/quick-perf/doc/wiki/JVM-annotations#ExpectNoJvmIssue).
For example, the following warning is reported with Maven 3.2.5:
```
Rule: Thrown Exceptions
Severity: WARNING
Score: 97
Message: The program generated 20 482 exceptions per second during 26,722 s starting at 
03/09/19 17:08:31.
```

# Acknowledgments
Many thanks to Hervé Boutemy for his help and support to start this project.

# License
[Apache License 2.0](/LICENSE.txt)
