---
title:  "Building a sampling profiler with 30-year-old technology"
author: cldellow
tags:
  - jvm
  - bash
  - profiler
  - sampling profiler
---

_This was originally published on the [Sortable dev blog](https://dev.sortable.com/)._

_2018-06-24: Someone has drawn my attention to the [Poor Man's Profiler](https://poormansprofiler.org/), which is a work of art._

Imagine that you've inherited this program:

```scala
import scala.util.Random

object SlowApp {
  def main(args: Array[String]): Unit = {
    (0 until 1000).par.foreach { _ => writeData(processData(readData)) }
  }

  private def readData = Stream.continually(Random.nextInt.abs % 100).take(10000).toArray

  private def processData(data: Array[Int]) = bubblesort(data)

  private def writeData(data: Array[Int]) = println(data.sum)

  private def bubblesort(data: Array[Int]): Array[Int] = {
    (0 until data.length).foreach { i =>
      (i + 1 until data.length).foreach { j =>
        if(data(i) > data(j)) {
          val tmp = data(i)
          data(i) = data(j)
          data(j) = tmp
        }
      }
    }

    data
  }
}
```

It's running in production and its runtime performance isn't great. You poke around with [`htop`](http://hisham.hm/htop/) and observe that all of the CPU cores are running at 100%. Maybe the app is thrashing in GC hell? You investigate with [`jstat -gcutil`](https://docs.oracle.com/javase/8/docs/technotes/tools/unix/jstat.html#sthref292) and find that it's not. Argh. Where's the hotspot?

You could look in the logs for informative messages--but this program doesn't log anything. D'oh!

If you could change the source code, you could add some log messages to time the various methods.

If you could restart the app, you could enable the [`hprof`](http://docs.oracle.com/javase/7/docs/technotes/samples/hprof.html) profiler that comes with the JVM to produce an informative summary of where CPU time was spent.

If you could create a tunnel to expose JMX on the server (possibly restarting the server to change its configuration), you could use JVisualVM's excellent sampling profiler as documented on [Lookout's technical blog](http://hackers.lookout.com/2014/06/profiling-remote-jvms/).

But what if none of the above is true, and you need to diagnose this performance issue in a non-invasive manner? I recently found myself in this situation. Here's my solution: a sampling profiler.

"But wait," I hear you cry. "A sampling profiler? You may as well just take a random stab in the dark 20 times." If you disagree that sampling profilers are effective ways to track down performance issues, go read [this rebuttal](http://stackoverflow.com/questions/375913/what-can-i-use-to-profile-c-code-in-linux/378024#378024).

## The solution

Twenty lines of bash scripting gets you a bare bones sampling profiler that is easily portable and has minimal dependencies.

```bash
#!/bin/bash -e

PID=${1:?must specify pid}
SAMPLES=${2:-100}

# Collect some stack traces.
rm -rf stack{,s}
mkdir stack{,s}
for i in $(seq 1 $SAMPLES); do
  jstack $PID > stacks/$i
  sleep 0.1
done

# Each stack trace file has a stack for every thread; filter the stacks
# to get only the runnable stacks, and put them in their own file.
awk '
/^$/          { lines = 0; next; }
/ RUNNABLE/   { stack = stack + 1; lines = 10; next; }
lines > 0     { lines = lines - 1; print $0 > "stack/" stack }' stacks/*

# List stacks ordered by how frequently they occured.
md5sum stack/* | sort | uniq -c --check-chars 32 | sort -n
```

There's not much to it: the code loops a few times to sample thread stacks every 100ms.

Next, we need to filter the stacks a little bit. Not every thread is busy doing real work. The main thread, for example, looks like:

```
"main" prio=10 tid=0x00007fd70800a000 nid=0x3861 in Object.wait() [0x00007fd7100c6000]
   java.lang.Thread.State: WAITING (on object monitor)
        at java.lang.Object.wait(Native Method)
        - waiting on <0x0000000705c0a770> (a scala.collection.parallel.AdaptiveWorkStealingForkJoinTasks$WrappedTask)
```

This makes sense: the main thread has to wait for the parallel computations to finish before terminating. In a typical server process you'd also have a bunch of idle GC and network threads that are waiting for data to become available before they can be read into a thread.

To help filter these stacks, we use Awk to create a separate file for the first 10 lines of every stack that's in the `RUNNABLE` state.

Now we have a bunch of files that each represent a vote for The Slowest Function Award. To tally them up, we need to write a program to compare the stacks, see which ones are identical, and count votes accordingly.

Luckily, the GNU coreutils have us covered. `md5sum` will tell us which files are identical, `uniq -c` will tally votes
and `sort -n` will rank them.

## The result

Let's give our new profiler a try on the sample program!

After taking 100 samples at 100ms intervals, it prints out:

```
      1 f28002fc7b8764980eb573809afac954  stack/964
      2 4b75085fae2e8e52da7ed218a5077a99  stack/1041
      2 c9031311464cd43cf33c8b387be46265  stack/626
      3 85fc2ed7dd0f4d03dfa91a6b8ca1ed46  stack/1162
    780 9a1c962f53aa008d52a7f5f1ab01b941  stack/100
```

Ouch--that's a pretty clear statement that whatever was in `stack/100` was doing a lot of work. Let's have a look:

```
        at scala.collection.immutable.Range.foreach$mVc$sp(Range.scala:168)
        at SlowApp$$anonfun$bubblesort$1.apply$mcVI$sp(SlowApp.scala:16)
        at scala.collection.immutable.Range.foreach$mVc$sp(Range.scala:166)
        at SlowApp$.bubblesort(SlowApp.scala:15)
        at SlowApp$.SlowApp$$processData(SlowApp.scala:10)
        at SlowApp$$anonfun$main$1.apply$mcVI$sp(SlowApp.scala:5)
        at scala.collection.parallel.immutable.ParRange$ParRangeIterator.foreach(ParRange.scala:91)
        at scala.collection.parallel.ParIterableLike$Foreach.leaf(ParIterableLike.scala:972)
        at scala.collection.parallel.Task$$anonfun$tryLeaf$1.apply$mcV$sp(Tasks.scala:49)
        at scala.collection.parallel.Task$$anonfun$tryLeaf$1.apply(Tasks.scala:48)
```

Unsurprisingly, it turns out that bubblesort is the bottleneck. Easily fixed!

Although this example was trivial, the approach itself scales well to real-world problems. Give it a try the next time you find your server melting.
