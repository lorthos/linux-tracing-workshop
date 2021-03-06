### CPU Sampling with `perf` and Flame Graphs

In this lab, you will experiment with CPU profiling of a simple application. The techniques you learn will be applicable to application profiling as well as profiling the entire system, will apply to user-mode and kernel-mode code, and will apply to any language or framework that has `perf` symbol support through the `/tmp/perf-$PID.map` convention, such as Java and Node.js.

- - -

#### Task 1: Compile the Profiled Application

The application you're about to profile is a very simple command-line tool that counts prime numbers in a given range, and uses [OpenMP](http://www.openmp.org) to parallelize the computation. But first, let's build it and make sure everything works. Navigate to the directory that contains [primes.c](primes.c) and build it using the following command:

```
$ gcc -g -fno-omit-frame-pointer -fopenmp primes.c -o primes
```

Now, experiment with multiple executions of this application with varying numbers of threads by running the following command:

```
for i in `seq 1 2 16`; do \
  OMP_NUM_THREADS=$i bash -c 'time ./primes'; \
done
```

Elapsed time should stabilize when you create enough threads to keep all the cores saturated throughout the entire run. Because the distribution of work between threads is not ideal, on an N-core system you will need more than N threads to keep them all busy.

- - -

#### Task 2: Profile with `perf`

It's time to find out what's taking so long to count our primes. Use the following command to get `perf`'s opinion. You will need a root shell for this.

```
# export OMP_NUM_THREADS=16
# perf record -g -F 997 -- ./primes
```

> In the preceding command, the `-g` switch indicates that you want the call stack to be captured; the `-F` switch lets you set the rate of sampling for events -- 997 times times per second in this case.
> The trace will include events aggregated across all threads spawned by `primes`. To include events from other processes while the program is running, specify `-a`, and if you really want
> to trace only a single thread within a process, you can use use `-t`.

To figure out where the bottleneck is, inspect `perf`'s report:

```
# perf report --stdio
```

It seems that we're spending a whole lot of time in the `is_divisible` function. If you want to know where exactly, you can use the following command:

```
# perf annotate
```

> Note that, as discussed in class, there may be skid in the displayed results. Notably, the specific instruction that seems to be taking most of the time might actually be masquerading for the previous or following instruction (or couple of instructions).

- - -

#### Task 3: Extract a Flame Graph

Now that you have the basic picture of what's happening in the application, it would be interesting to inspect everything else. The perf report is quite long, though:

```
# perf report --stdio | wc -l
1286
```

> NOTE: Your report might be a lot shorter; this is just an example. In a real-world application, a report that is hundreds of thousands of lines long will not be unusual. In this simple demo, your report might be just a handful of lines.

So instead of reading long reports, let's take flame graphs for a spin. If you haven't already, clone the [FlameGraph](https://github.com/BrendanGregg/FlameGraph) repository, and then run the following commands to post-process the perf report:

```
# perf script > primes.stacks
# FlameGraph/stackcollapse-perf.pl primes.stacks > primes.collapsed
# FlameGraph/flamegraph.pl primes.collapsed > primes.svg
```

Open the resulting flame graph and enjoy the results. It is now immediately evident that there are no significant stack tree branches other than the `is_prime` calling `is_divisible` branch that we have already seen.

- - -

#### Task 4: Profiling Java Code

In this section, you are going to profile some Java code and generate flame graphs for that as well. To help `perf` figure out the symbols (function names) for your Java code, you will use [perf-map-agent](https://github.com/jrudolph/perf-map-agent), which attaches to your Java process and generates map files that map dynamically-generated Java machine code to class and method names. `perf-map-agent` can also run the whole recording session for you, through a set of handy scripts such as
`perf-java-record-stack` and `perf-java-report-stack`.

First, run the [Java application](slowy/Slowy.java) that we're about to profile. It is a Java version of the same prime-counting app:

```
$ java -XX:+PreserveFramePointer slowy/App
```

Do not hit ENTER yet. Instead, in another (non-root) shell, run `jps` to find the process id for the Slowy app, and then run the collection tool. After you run `perf-java-report-stack`, hit ENTER in the Java app's console so that the collection tool records some meaningful work.

```
$ jps
2144 Jps
2132 App
$ ./perf-java-report-stack 2132   # use the pid from the previous step
                                  # hit ENTER in the java console
```

If everything went well, you should see the `perf report` ncurses UI, showing the bottlenecks in the Java program. These will likely be `App::main` and `App::isPrime`, because the smaller methods were optimized away (inlined). You can repeat the experiment and run Slowy with the `-XX:-Inline` switch to prevent this optimization and obtain more accurate results that include the `App::isDivisible` method.

To generate flame graphs from this report, you need the perf.data file created by `perf`. By default, `perf-java-report-stack` places it in `/tmp/perf-PID.map`, where _PID_ is the process ID of your Java process. You can then run the following command to generate a flame graph:

```
$ perf script | FlameGraph/stackcollapse-perf.pl | FlameGraph/flamegraph.pl --colors=java > java.svg
```

Note the `--colors=java` parameter, which instructs the flamegraph.pl script to colorize functions according to their type: Java methods, native methods, and kernel methods will all get their own distinctive color palettes.

- - -

#### Task 5: Profiling Node.js Code

In this section, you are going to profile a Node.js application. To help `perf` figure out the symbols (function names) for your Node.js code, you will need to run the Node process with the `--perf_basic_prof` or `--perf_basic_prof_only_functions` command-line switches. These instruct the Node runtime to generate a map file for each JIT-compiled method, which maps the method address to the method name that `perf` can parse.

Navigate to the `nodey` directory. If you haven't yet, you should make sure all required packages are installed by running `npm install`. Then, run the following command to start our simple Node application with perf map support:

```
$ ./run.sh perf
```

Now, in a root shell, start recording CPU samples using `perf`:

```
# perf record -F 97 -p $(pgrep -n node) -g
```

In the first shell, run the following command to execute a simple benchmark of one of our Node application's endpoints, which is consuming lots of CPU:

```
$ ab -c 10 -n 100 -m POST 'http://localhost:3000/users/auth?username=foo&password=wrong'
```

When the benchmark completes, go back to the root shell and stop the recording by hitting Ctrl+C. Change the permissions on the perf.data file so that you can view it as a standard user:

```
# chown fedora perf.data
```

Now, go back to the first shell. You can view the report in text format by running:

```
$ perf report -i /path/to/perf.data
```

Or, you can generate a flame graph from the report, by using the following command:

```
$ perf script -i /path/to/perf.data | FlameGraph/stackcollapse-perf.pl | FlameGraph/flamegraph.pl > node.svg 
```

- - -

#### Bonus: Some More Flame Graphs

For some more fun, experiment with collecting CPU samples and then generating flame graphs while the following commands are running:

1. `sha1sum /dev/zero`
1. `dd if=/dev/zero of=/dev/null bs=100K count=50K`

You might get a lot of `[unknown]` frames because of missing debuginfo. Try to obtain debuginfo for the missing components and then try again. If there's no choice, you can always build from source with the `-g` option to make sure debuginfo will be available.
