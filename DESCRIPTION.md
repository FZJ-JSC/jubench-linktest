LinkTest
========

TLDR: With JUBE, add system specific definitions to `benchmark/jube/LinkTestMain.xml` and run `jube run benchmark/jube/LinkTestMain.xml`.

## About

LinkTest is a communication benchmarking tool that tests point-to-point connections in serial or parallel. It scales to a very large numbers of hosts (tested using 1 800 000 MPI tasks on JUQUEEN, Blue Gene/Q). It supports the benchmarking of MPI, TCP, UCP, IB Verbs, PSM2, and NVLink bridges via CUDA. Output of the program is a full timing communication matrix of the message-transmission times for all pairs of hosts written to a SION file in parallel. A standard out log that summarizes the results is also provided. Tools are provided to read the SION file into Python and generate A4-page PDF reports. 

More details can be found in the [LinkTest Wiki](https://gitlab.jsc.fz-juelich.de/cstao-public/linktest/-/wikis/LinkTest).

## Source

Archive name: `linktest-bench.tar.gz`.

In the archive, the LinkTest source code is available in `src/linktest/`. It is equivalent to commit `3d7dac74` of the LinkTest source.

## Requirements

LinkTest needs a C++ compiler (like GCC's `g++`) and an MPI implementation (like OpenMPI) for operation.

In addition, SIONlib is required for result files and Python (3.8.4+; including Numpy and matplotlib packages) for generating PDF reports. Furthermore, the benchmark is deeply integrated with JUBE; hence, we strongly recommend using JUBE.

LinkTest has the following requirement to the system and its environment:

* Every node must have a uniquely identifiable hostname.
* Every node must be POSIX compliant and a bash shell must be available.
* The terminal used for submission and possible post-processing should ideally support UTF-8.

## Build

_Especially for LinkTest, we recommend using JUBE for compilation and execution. See below._

For detailed tutorials also see [https://gitlab.jsc.fz-juelich.de/cstao-public/linktest/-/wikis/Build](https://gitlab.jsc.fz-juelich.de/cstao-public/linktest/-/wikis/Build)

### Main Benchmark

LinkTest needs to be build with MPI for the benchmark:

```bash
cd src/linktest/benchmark
make -j 12 HAVE_MPI=1
```

### LinkTest Reporting Tool

To install `linktest-report`, LinkTest's Python-based reporting tool, execute the following from the `src/linktest/benchmark/` directory:

```bash
cd ../python
python3 -m pip install . --use-feature=in-tree-build
cd ..
```

Installation will be either global, if possible, otherwise user-specific.  
Build failures might be related to missing Python headers. We use workarounds like the following

```
export CPATH=/p/software/juwels/stages/2022/software/SciPy-bundle/2021.10-gcccoremkl-11.2.0-2021.4.0/lib/python3.9/site-packages/numpy/core/include:$CPATH
```

### JUBE

JUBE can build and execute the benchmark automatically. The provided JUBE script `LinktestMain.xml` works out of the box on JSC systems. To ease configuring it for new systems, the script already holds commented-outed placeholders, omitting the dictionary look ups.

The JUBE script incorporates a `Compile` step which loads all necessary modules and compiles the benchmark. The step is automatically included when running the script, see below in Section Execution.

## Execution

### Command Line

For detailed tutorials, also see [_Usage_ in the LinkTest Wiki](https://gitlab.jsc.fz-juelich.de/cstao-public/linktest/-/wikis/Usage).

```
srun --cpu-bind=v,rank_ldom --distribution=block \
    Compile/benchmark/linktest \
        --mode mpi \
        --num-warmup-messages 100 \
        --num-messages 1000 \
        --size-messages 16777216 \
        --bidirectional\
        --bisection
```

With Slurm, the bisection constraints of the benchmark can be realized with heterogeneous jobs and the `--constraint` parameter, see the JUBE script for an example.

### LinkTest Reporting Tool

For detailed tutorials, also see [the _LinkTest-Report_ page of the LinkTest Wiki](https://gitlab.jsc.fz-juelich.de/cstao-public/linktest/-/wikis/LinkTest-Report).

The LinkTest PDF report will only be used for qualitative evaluation but can also be helpful for the submitter, especially if you are analyzing performance.
The `linktest-report` executable should be in your path after installation. If you used JUBE, the SION file will be in a directory similar to `<path_to_runs_folder>/runs/000000/000001_BisectionTest/work/pingpong_results_bin.sion`.

```bash
linktest-report -i <path_to_file>/pingpong_results_bin.sion -o report.pdf
```

### JUBE

Using JUBE, the benchmark is automatically built and run. Beyond the JSC-system-specific options, further opportunities for modification are, for example, `taskspernode` (which is currently set to the number of HCAs per node), and processor core affinity (which is currently set such that cores with HCA affinity are chosen). At JSC, Slurm with `--cpu-bind=rank_ldom --distribution=block` starter arguments is used to achieve best performance. The `--constraint` argument is used to ensure proper bisection.

Compile and run the benchmark like the following, giving optional tags (see below)

```
jube run benchmark/jube/LinkTestMain.xml --tag <tags>
```

See the following table for available tags:

| Tag    | Meaning                                                 |
| ------ | ------------------------------------------------------- |
| `test` | Run small test with 2 nodes and few number of messages  |
| `pdf`  | Additionally build linktest-report, generate PDF report |

The default is to run the full system test without PDF generation.

_Note that the build process of the `linktest-report` binary used for PDF generation is somewhat unstable due to the interaction of Python and C code. There is a showcase of a common workaround in `LinktestMain.xml` in `<step name="CompileLinktestReport" ...`_

After the job is finished, show results with

```
jube continue runs
jube result runs
```

## Rules, Workflow

1. **Perform a bidirectional bisection test**: Use at least 25 messages (3 warmup) of size 16 MiB, with LinkTest options `--mode mpi --num-warmup-messages 3 --num-messages 25 --size-messages 16777216 --bisection --bidirection`
2. Use as many MPI tasks as there are HCAs in the system. If the number of HCAs is uneven, use one MPI task less.
3. The first of the bisecting halves will contain MPI tasks 0 to (n/2 - 1), the second half will contain n/2 to n. Run the test across your network bottleneck. If there is no theoretical bottleneck, i.e. because you are using a full fat tree or a k-complete topology, please set up the benchmark such that communication must go across top-level switches, if there are any (for a k-complete topology disregard this as there cannot be any theoretical bottlenecks – in that case, set up your configuration as you please within the k-complete topology). This requires you to equally distribute ranks on either half of the bottleneck (for example through using `#SBATCH --constraint` in the `additional_job_config` parameter). Please also ensure correct number of tasks per node and their placement (for example through using the `args_starter` and `taskspernode` parameters).

## Results

JUBE will automatically generate the results table (using `jube result <path_to_runs_folder>`). If JUBE is not employed for execution, the values to report can be extracted from the raw LinkTest output, as per the following guidelines.

Each step of LinkTest will print a log line like this

```
Parallel Results for step 3: min:   11.64910 us (   20.9579 GiB/s) avg:   11.68455 us (   20.8943 GiB/s) max:   11.77200 us (   20.7391 GiB/s) avg. bisection bw:    83.5783 GiB/s
Aggregated Results: min:   11.53511 us (   21.1650 GiB/s) avg:   11.68784 us (   20.8884 GiB/s) max:   12.01200 us (   20.3247 GiB/s) avg. bisection bw:    83.5584 GiB/s
Timing Summary: Elapsed time:   24.52618 ms (   8.17539 ms/step). Estimated time remaining for iteration:    8.17539 ms (1 steps). Estimated total time remaining:    8.17539 ms
```

Considering all `Parallel Results for step X` entries, note the `avg. bisection bw`. Of these, the minimum and maximum value are to be reported as _Min_ and _Max_ Bisection Bandwidth (see below).  
From the last `Aggregated Results` line, the value of `avg. bisection bw` is to be reported as _Avg_ Bisection Bandwidth (see below).

In addition, the values for `min`, `max`, and `avg` of the `Aggregated Results` line should be reported for qualitative evaluation.

### Example (JUWELS Booster)

Below are example results from 4 JUWELS Booster cells consisting of 48 nodes (as an example, 2 nodes omitted to realize an incomplete cell => 4×48-2 = 190 in total), each equipped with four 200 Gb/s (23.28 GiB/s) InfiniBand cards connected to a full-fat-tree network within the cell. 4 LinkTest processes were run on the 4 different CPU cores closest to the 4 InfiniBand cards. The bisection was done so that each half consisted of 2 cells.

Configuration:

| Messages |   Size   |            Options            | Nodes | Tasks per Nodes |
|----------|----------|-------------------------------|-------|-----------------|
|     1000 | 16777216 | `--bidirectional --bisection` |   190 |               4 |

Bisection Bandwidth Results:

|  Unit |    Min |    Avg |     Max |
|-------|--------|--------|---------|
| TiB/s | 8.2409 | 9.2899 | 11.3783 |

## Commitment

Given the above-mentioned rules (≧ 25 messages, 3 warm-up, 16 MiB, etc.) and using as much MPI tasks as HCA adapters (or equivalent high-speed interconnect adapters) in the system, the minimum, average, and maximum bisection bandwidth is to be reported, as outlined above.  
The minimum value is used for the quantitative evaluation, the other values are used for the qualitative evaluation. Also to be committed is the LinkTest report and the three _Aggregated Results_ values of the LinkTest output (`min`, `max`, `avg`), which are used for the qualitative evaluation.
