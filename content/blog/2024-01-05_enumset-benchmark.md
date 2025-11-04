+++
title = "Benchmarking the cost of Javas' EnumSet - A Second Look"
date = 2024-01-05
authors = ["Thomas Kinnen"]
+++

A few months ago, I read Chris Wellons' ["The cost of Java's EnumSet"](https://nullprogram.com/blog/2021/04/23/). I really enjoyed the read, but as a long time Java developer, I was left with few questions and thoughts:
1. "Is Java's `EnumSet` really that slow? 3 orders of magnitude feels like a way too big margin."
1. "Writing benchmarks in Java is inherently tricky, especially for small and fast operations. We should probably use a dedicated tool like [JMH](https://github.com/openjdk/jmh) for that."

Now I finally had some time on my hands to go and explore these thoughts and try to fill the void with some cold hard data. This post will first explain a few important aspects of profiling java and then discuss the results of our specific benchmark, by answering our research questions:

# Benchmark Goals & Questions

I want to see if I can reproduce Chris' results and answer a few additional questions at the same time:
1. Is a bit field really a factor 1000 faster than an `EnumSet`?
1. How fast are the create/equals/add/remove operations individually?
1. How easily can we "enable" inlining and how big is the difference?
1. Is using `Set.of` faster/slower than the other options? To me this would be the most modern way of creating a Set of a few objects, if you don't use `EnumSet` directly: `Set<Flag> setOfA = Set.of(Flag.A, Flag.B, Flag.E);`. Using this code, we leave picking the 'correct' implementation entirely up to the API.

# Benchmarking in Java

In order to answer the first question, we'll have to look into using JMH for benchmarking first. Since Java runs on the JVM, a runtime environment that does all sorts of optimizations at runtime, based on previous behavior of the code, it is quite difficult to write a benchmark for small operations, as these will likely be optimized by inlining variables or eliminating entire code regions because they are either dead or can be replaced by a constant value.

Take the example from Chris' post, that he correctly points out at being completely optimized away:

```java
static void benchmarkBitField() {
    System.gc();
    long beg = System.nanoTime();
    int a = A | B | G;
    for (int i = 0; i < 1_000_000_000; i++) {
        int b = A | B | G;
        assert a == b;
    }
    long end = System.nanoTime();
    System.out.println("bitField\t" + (end - beg)/1e9);
}
```

After a while, the JVM will figure out that the only side effect this method has, is printing the time passed. How does it do it? The steps are *roughly* as follows:

1. It figures out that `a` and `b` are constants, so it does not need to recreate `b` on every loop
1. It figures out that `a == b` is always `true`
1. It figures out that the loop does the same thing without any side effects *1_000_000_000* times, so it can be removed completely. 
1. It figures out that `end` can be inlined.

To the JVM the method now basically looks like this:

```java
static void benchmarkBitField() {
    System.gc();
    long beg = System.nanoTime();
    System.out.println("bitField\t" + (System.nanoTime() - beg)/1e9);
}
```

As you can tell from Chris' results, the JVM is pretty smart here, it only takes the JVM two method executions, until the third execution is already fully optimized. Parts of the optimization are probably already being done during the compilation as well.

## Avoiding inlining using JMH's helpers: `Blackhole` and `State` objects
To really compare the performance differences of the `a==b` statement, we want to avoid it being optimized away in our benchmark. In order to avoid inlining, frameworks such as JMH provide a number of mechanism, such as a `Blackhole` instance, as well as `State` objects. You can read more about JMH, these objects and how to write good benchmarks in the great write-up by *Jakob Jenkov* here: [JMH - Java Microbenchmark Harness](https://jenkov.com/tutorials/java-performance/jmh.html).

Let's take a look at a JMH adaption of Chris' benchmark:

```java
@Benchmark
public void benchmarkBitFieldEquals(BitFieldState state, Blackhole bh) {
    bh.consume(state.a == state.b);
}

@State(Scope.Thread)
public static class BitFieldState {
    public int a = A | B | E;
    public int b = C | D | F;
}
```

To avoid over-eager inlining by the JVM, we use both a `State` object, that holds the elements `a` and `b` that we want to compare and a `Blackhole` that consumes the value. To ensure that this actually makes a difference, I added a non-inline-protected version:

```java
@Benchmark
public void benchmarkBitFieldEqualsInline(Blackhole bh) {
    int a = A | B | E;
    int b = C | D | F;
    bh.consume(a == b);
}
```

The results are pretty clear. The inlinable version is almost twice as fast (the Blackhole doesn't seem to make a difference according to my test, so adding the `BitFieldState` into the mix is really necessary):
```
EnumSetBenchmark.bitFieldEquals          1582210639.358 ± 33561406.255  ops/s
EnumSetBenchmark.bitFieldEqualsInline    2634237498.969 ± 37811030.834  ops/s
```

With that out of the way, let's dive into the actual benchmark!

# Benchmark Results

Let's take a look at the results. You can find the entire [benchmark implementation](https://github.com/nihathrael/enumset_benchmark/blob/main/app/src/jmh/java/de/kinnen/enumset_benchmark/EnumSetBenchmark.java) on Github, so that you can take a look at the code and rerun the benchmark yourself: [https://github.com/nihathrael/enumset_benchmark](https://github.com/nihathrael/enumset_benchmark).

**Results**:
```
Benchmark                                Mode  Cnt           Score          Error  Units
EnumSetBenchmark.bitFieldAdd             thrpt    5  1122953845.662 ± 14487321.908  ops/s
EnumSetBenchmark.bitFieldCreation        thrpt    5  1292682130.158 ± 33722811.879  ops/s
EnumSetBenchmark.bitFieldCreationInline  thrpt    5  2640638042.506 ± 37382005.672  ops/s
EnumSetBenchmark.bitFieldEquals          thrpt    5  1582210639.358 ± 33561406.255  ops/s
EnumSetBenchmark.bitFieldEqualsInline    thrpt    5  2634237498.969 ± 37811030.834  ops/s
EnumSetBenchmark.bitFieldRemove          thrpt    5  1279302750.101 ± 22906668.356  ops/s
EnumSetBenchmark.enumSetAdd              thrpt    5   201261050.245 ±  3047021.038  ops/s
EnumSetBenchmark.enumSetCreate           thrpt    5   215090629.176 ±   827035.414  ops/s
EnumSetBenchmark.enumSetCreateInline     thrpt    5   239108143.810 ±  2845003.880  ops/s
EnumSetBenchmark.enumSetEquals           thrpt    5   586193130.841 ±  5057355.446  ops/s
EnumSetBenchmark.enumSetEqualsInline     thrpt    5   500615966.717 ±  3552721.301  ops/s
EnumSetBenchmark.enumSetRemove           thrpt    5   204125597.157 ±  1477564.086  ops/s
EnumSetBenchmark.hashSetCreate           thrpt    5    21140166.919 ±   331772.905  ops/s
EnumSetBenchmark.hashSetEquals           thrpt    5    53184863.238 ±   289455.786  ops/s
EnumSetBenchmark.setOfCreate             thrpt    5    26619186.610 ±   370502.131  ops/s
EnumSetBenchmark.setOfEquals             thrpt    5    54846987.946 ±  1022244.569  ops/s
```

# Discussion
Let's dive into the results and answer one question at a time:

##### 1. Is a bit field really a factor 1000 faster than an `EnumSet`?
No, it looks like, without the JVM being able to optimize over eagerly, they are about *3-6x* as fast. That is still a lot, but at least to me, that sounds much more reasonable and in line with what I would have expected (after all, the `RegularEnumSet`, used for `Enums` up to a size of 64, uses a bit field internally, so, as Chris correctly outlined in his post, we only expect overhead for the `EnumSet` instances and some additional boilerplate such as type checks within the implementation). To me it means that using a bit field to represent a set of enums can be a viable approach if you really need every last millisecond (or save tons of RAM, an aspect that we haven't looked at all yet), but for nearly every normal use case, using `EnumSet` should be just fine.

##### 2. How fast are the create/equals/add/remove operations individually?

For bit fields, all operations are pretty much the same speed and achieve the same ops/s. Since most of these are very simple bit shift or integer comparisons, that is not super surprising. 
`EnumSet` is the second fastest option, which is about ~3x slower for equals operations and about 6x slower on create/add and remove, compared to the pure bit field.
`HashSet`s are quite a bit slower, they are about a factor 10x slower than `EnumSets` for all operations. To me this means that using the specialized `EnumSet` is definitely worth it, as it basically comes at no cost other than having to remember to use it.

##### 3. How easily can we "enable" inlining and how big is the difference?
Looking at the inline version of the creation and equals methods also vividly demonstrates how much performance JVM optimizations provide, if they can be applied effectively. The inlinable versions of the bit fields are ~2x faster than the non-inlined version. In my example we also only inline one or two operations, not entire loops like in Chris' example, where it was very effective, leading to an even higher speedup. 
For the `EnumSet`s the inlining does not seem to happen, at least on our contrived example, the ops/s are pretty much the same. This underlines Chris' point, that these are hard for the JVM to optimize, even more so in more complex logic, like in his example.

##### 4. Is using `Set.of` faster/slower than the other options?

Using `Set.of` is a tiny bit faster than using a `HashSet`, but still ~10x slower than `EnumSet`s. This finding surprised me the most, as I would have expected the API to be smart enough to pick an `EnumSet` internally, if I explicitly add Enum type objects. So the take away, again, is to try to remember to use `EnumSet` whenever dealing with sets of enums and to not trust the standard API to be smart enough to figure it out. 

# Summary

Overall, bit fields are a 3-6x faster than `EnumSet`s in my benchmark and are about ~60x faster than `HashSet`s or using `Set.of(e1, e2, e3)`. As a general rule of thumb, you should and can always use `EnumSet` when dealing with sets of enums, unless you have a very clear performance problem pointing at the `EnumSet` directly. Of course, as always in Java, it comes at an overhead of memory use, compared to bare bit fields, but is nicer to read and maintain, as well as providing better compiler support (e.g. in switch statements, where the compiler will warn you if are missing a case).
Big thanks to Chris for opening the discussion on this topic and I hope I could provide some new insights and numbers for you to make your own decisions on what to use for your problem. 

If you have any questions or remarks, feel free to contact me via one of my [socials](/#socials)!











