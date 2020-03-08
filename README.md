# Criterium

Criterium measures the computation time of an expression.  It is
designed to address some of the pitfalls of benchmarking, and benchmarking on
the JVM in particular.

This includes:

  * statistical processing of multiple evaluations
  * inclusion of a warm-up period, designed to allow the JIT compiler to
    optimise its code
  * purging of gc before testing, to isolate timings from GC state prior
    to testing
  * a final forced GC after testing to estimate impact of cleanup on the
    timing results

## Installation

### Leiningen

Add the following to your `:dependencies`:

```clj
[criterium "0.4.5"]
```

### Maven

```xml
<dependency>
  <groupId>criterium</groupId>
  <artifactId>criterium</artifactId>
  <version>0.4.5</version>
</dependency>
```

## Usage

The top level interface is in `criterium.core`.

    (use 'criterium.core)

Use `bench` to run a benchmark in a simple manner.

```
(bench (Thread/sleep 1000))
 =>
                   Execution time mean : 1.000803 sec
          Execution time std-deviation : 328.501853 us
         Execution time lower quantile : 1.000068 sec ( 2.5%)
         Execution time upper quantile : 1.001186 sec (97.5%)
```

By default bench is quiet about its progress.  Run `with-progress-reporting` to
get progress information on `*out*`.

```clj
(with-progress-reporting (bench (Thread/sleep 1000) :verbose))
(with-progress-reporting (quick-bench (Thread/sleep 1000) :verbose))
```

Lower level functions are available, that separate benchmark statistic
generation and reporting.  `benchmark` and `quick-benchmark` return
Clojure maps containing multiple measurements of running your
expression, and details about your JVM version and operating system.

```clj
(report-result (benchmark (Thread/sleep 1000) {:verbose true}))
(report-result (quick-benchmark (Thread/sleep 1000)))
```

Note that results are returned to the user to prevent JIT from recognising that
the results are not used.

| "Thorough" benchmarks | "Quick" benchmarks | Notes |
| --------------------- | ------------------ | ----- |
| `bench`     | `quick-bench`     | Returns `nil`.  Prints human-readable output. |
| `benchmark` | `quick-benchmark` | Returns map of measurement data.  Prints nothing.  Printing can be done separately using `report-result`. |
| at least 70 sec | at least 8 sec | Minimum time to run one benchmark, with default parameter values (see below for their values and how to change them). |

The table below shows several options that can be specified when
running benchmarks, and their default values if you do not specify
one.

| "Thorough" benchmarks | "Quick" benchmarks | Parameter |
| --------------------- | ------------------ | --------- |
| 10 sec |   5 sec | `:warmup-jit-period`, units: nanoseconds |
|  1 sec | 0.5 sec | `:target-execution-time`, units: nanoseconds |
| 60     | 6       | `:samples`, units: number of samples |
| 70 sec = 10 + 60 x 1 | 8 sec = 5 + 6 * 0.5 | Approximate time required for one execution, using default parameters |
| `true` | `true` | `:gc-before-sample`, boolean. See Note 1 below. |

Notes on `:gc-before-sample`:

When `:gc-before-sample` is `true`, criterium calls
[`java.lang.System/gc`](https://docs.oracle.com/javase/8/docs/api/java/lang/System.html#gc--)
`:samples` times, once before each sample is taken.  The intent is
that each sample will be more likely to start with all
no-longer-referenced garbage objects reclaimed in memory (this is not
guaranteed, since the documented behavior of the `gc` method is that
it "suggests that the Java Virtual Machine expend effort").  Note that
this call is done once before each sample, not once before each
execution of your benchmarked expression.  The time for these garbage
collection runs is not directly included in the measurements. but
depending upon which garbage collector is in use by the JVM, it may
affect the measurements.

Note 1: All criterium versions 0.4.0 through 0.4.5 have a bug where
the value of the `:gc-before-sample` option is always `true`, and it
cannot be changed to `false` (without [fixing the
bug](https://github.com/hugoduncan/criterium/pull/49)).

Below are examples of benchmark runs that use non-default parameter
values.  `:verbose` can be omitted for shorter output.

```clojure
(use 'criterium.core)

;; s-to-ns is simply 1e9, useful for conversion from seconds to nanoseconds

(bench (Thread/sleep 25)
       :verbose
       :warmup-jit-period (* 7 s-to-ns)
       :target-execution-time (* 2 s-to-ns)
       :samples 10)
;; prints human-readable results, not shown here

(quick-bench (Thread/sleep 25)
             :verbose
             :warmup-jit-period (* 7 s-to-ns)
             :target-execution-time (* 2 s-to-ns)
             :samples 10)
;; prints human-readable results, not shown here

(def r1 (benchmark (Thread/sleep 25)
                   {:warmup-jit-period (* 7 s-to-ns)
                    :target-execution-time (* 2 s-to-ns)
                    :samples 10}))
;; returns map, not shown here
(report-result r1 :verbose)
;; prints human-readable results, not shown here

(def r2 (quick-benchmark (Thread/sleep 25)
                         {:warmup-jit-period (* 7 s-to-ns)
                          :target-execution-time (* 2 s-to-ns)
                          :samples 10}))
;; returns map, not shown here
(report-result r2 :verbose)
;; prints human-readable results, not shown here
```

## Measurement Overhead Estimation

Criterium will automatically estimate a time for its measurement
overhead.  The estimate is normally made once per session, taking
about 17 to 18 seconds, and is available in the
`criterium.core/estimated-overhead-cache` var.

If the estimation is made while there is a lot of other processing
going on, then benchmarking quick functions may report small negative
times.  You can force a recalculation of the overhead by calling
`criterium.core/estimated-overhead!`.

If you want consistency across JVM processes, it might be prudent to
explicitly set `criterium.core/estimated-overhead!` to a constant
value.

## References

[API Documentation](http://hugoduncan.github.com/criterium/0.4/api/)
[Annotated Source](http://hugoduncan.github.com/criterium/0.4/uberdoc.html)

See [Elliptic Group](http://www.ellipticgroup.com/html/benchmarkingArticle.html)
for a Java benchmarking library.  The accompanying article describes many of the
JVM benchmarking pitfalls.

See [Criterion](http://hackage.haskell.org/package/criterion) for a Haskell
benchmarking library that applies many of the same statistical techniques.

## Todo

Serial correlation detection.
Multimodal distribution detection.
Use kernel density estimators?

## Releasing

To release, run the `release.sh` script.  This requires that you have
git-flow enabled your git repository with `git flow init`, and that
you have configured your
[credentials for clojars](https://github.com/technomancy/leiningen/blob/stable/doc/DEPLOY.md).

## YourKit

YourKit is kindly supporting open source projects with its full-featured Java
Profiler.

YourKit, LLC is the creator of innovative and intelligent tools for profiling
Java and .NET applications. Take a look at YourKit's leading software products:

* <a href="http://www.yourkit.com/java/profiler/index.jsp">YourKit Java Profiler</a> and
* <a href="http://www.yourkit.com/.net/profiler/index.jsp">YourKit .NET Profiler</a>.

## License

Licensed under [EPL](http://www.eclipse.org/legal/epl-v10.html)
