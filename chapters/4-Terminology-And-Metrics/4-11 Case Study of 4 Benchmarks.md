## Case Study {#sec:PerfMetricsCaseStudy}

Putting together everything we discussed so far in this chapter, we run four benchmarks from different domains and calculated their performance metrics. First of all, let's introduce the benchmarks.

1. Blender 3.4 - an open-source 3D creation and modeling software project. This test is of Blender's Cycles performance with BMW27 blend file. All HW threads are used. URL: [https://download.blender.org/release](https://download.blender.org/release). Command line: `./blender -b bmw27_cpu.blend -noaudio --enable-autoexec -o output.test -x 1 -F JPEG -f 1`.
2. Stockfish 15 - an advanced open-source chess engine. This test is a stockfish built-in benchmark. A single HW thread is used. URL: [https://stockfishchess.org](https://stockfishchess.org). Command line: `./stockfish bench 128 1 24 default depth`.
3. Clang 15 selfbuild - this test uses clang 15 to build clang 15 compiler from sources. All HW threads are used. URL: [https://www.llvm.org](https://www.llvm.org). Command line: `ninja -j16 clang`.
4. CloverLeaf 2018 - a Lagrangian-Eulerian hydrodynamics benchmark. All HW threads are used. This test uses clover_bm.in input file (Problem 5). URL: [http://uk-mac.github.io/CloverLeaf](http://uk-mac.github.io/CloverLeaf). Command line: `./clover_leaf`.

For the purpose of this exercise, we run all four benchmarks on the machine with the following characteristics:

* 12th Gen Alderlake Intel(R) Core(TM) i7-1260P CPU @ 2.10GHz (4.70GHz Turbo), 4P+8E cores, 18MB L3-cache
* 16 GB RAM, DDR4 @ 2400 MT/s
* 256GB NVMe PCIe M.2 SSD
* 64-bit Ubuntu 22.04.1 LTS (Jammy Jellyfish)

To collect performance metrics, we use `toplev.py` script that is a part of [pmu-tools](https://github.com/andikleen/pmu-tools)[^1] written by Andi Kleen:

```bash
$ ~/workspace/pmu-tools/toplev.py -m --global --no-desc -v -- <app with args>
```

The table {@tbl:perf_metrics_case_study} provides a side-by-side comparison of performance metrics for our four benchmarks. There is a lot we can learn about the nature of those workloads just by looking at the metrics. Here are the hypothesis we can make about the benchmarks before collecting performance profiles and diving deeper into the code of those applications.

* __Blender__. The work is split fairly equally between P-cores and E-cores, with a decent IPC on both core types. The number of cache misses per kilo instructions is pretty low (see `L*MPKI`). Branch misprediction contributes as a bottleneck: the `Br. Misp. Ratio` metric is at `2%`; we get 1 misprediction every `610` instructions (see `IpMispredict` metric), which is not bad, but is not perfect either. TLB is not a bottleneck as we very rarely miss in STLB. We ignore `Load Miss Latency` metric since the number of cache misses is very low. The ILP is reasonably high. Goldencove is a 6-wide architecture; ILP of `3.67` means that the algorithm utilizes almost `2/3` of the core resources every cycle. Memory bandwidth demand is low, it's only 1.58 GB/s, far from the theoretical maximum for this machine. Looking at the `Ip*` metrics we can tell that Blender is a floating point algorithm (see `IpFLOP` metric), large portion of which is vectorized FP operations (see `IpArith AVX128`). But also, some portions of the algorithm are non-vectorized scalar FP single precision instructions (`IpArith Scal SP`). Also, notice that every 90th instruction is an explicit software memory prefetch (`IpSWPF`); we expect to see those hints in the Blender's source code. Conclusion: performance is bound by FP compute with occasional branch mispredictions.
* __Stockfish__. We ran it using only one HW thread, so there is zero work on E-cores, as expected. The number of L1 misses is relatively high, but then most of them are contained in L2 and L3 caches. Branch misprediction ratio is high; we pay the misprediction penalty every `215` instructions. We can estimate that we get one mispredict every `215 (instructions) / 1.80 (IPC) = 120` cycles, which is very frequently. Similar to the Blender reasoning, we can say that TLB and DRAM bandwidth is not an issue for Stockfish. Going further, we see that there is almost no FP operations in the workload. Conclusion: Stockfish is a integer compute workload, which is heavily affected by branch mispredictions.

--------------------------------------------------------------------------
Metric           Core        Blender     Stockfish   Clang15-   CloverLeaf
Name             Type                                selfbuild
---------------- ----------- ----------- ----------- ---------- ----------
Instructions     P-core      6.02E+12    6.59E+11    2.40E+13   1.06E+12

Core Cycles      P-core      4.31E+12    3.65E+11    3.78E+13   5.25E+12

IPC              P-core      1.40        1.80        0.64       0.20

CPI              P-core      0.72        0.55        1.57       4.96

Instructions     E-core      4.97E+12    0           1.43E+13   1.11E+12

Core Cycles      E-core      3.73E+12    0           3.19E+13   4.28E+12

IPC              E-core      1.33        0           0.45       0.26

CPI              E-core      0.75        0           2.23       3.85

L1MPKI           P-core      3.88        21.38       6.01       13.44

L2MPKI           P-core      0.15        1.67        1.09       3.58

L3MPKI           P-core      0.04        0.14        0.56       3.43

Br. Misp. Ratio  E-core      0.02        0.08        0.03       0.01

Code stlb MPKI   P-core      0           0.01        0.35       0.01

Ld stlb MPKI     P-core      0.08        0.04        0.51       0.03

St stlb MPKI     P-core      0           0.01        0.06       0.1
    
Load Miss Lat.   P-core      12.92       10.37       76.7       253.89    

ILP              P-core      3.67        3.65        2.93       2.53

MLP              P-core      1.61        2.62        1.57       2.78

DRAM BW Use      All         1.58        1.42        10.67      24.57
     
IpCall           All         176.8       153.5       40.9       2,729

IpBranch         All         9.8         10.1        5.1        18.8

IpLoad           All         3.2         3.3         3.6        2.7

IpStore          All         7.2         7.7         5.9        22.0

IpMispredict     All         610.4       214.7       177.7      2,416

IpFLOP           All         1.1         1.82E+06    286,348    1.8

IpArith          All         4.5         7.96E+06    268,637    2.1

IpArith Scal SP  All         22.9        4.07E+09    280,583    2.60E+09

IpArith Scal DP  All         438.2       1.22E+07    4.65E+06   2.2

IpArith AVX128   All         6.9         0.0         1.09E+10   1.62E+09

IpArith AVX256   All         30.3        0.0         0.0        39.6

IpSWPF           All         90.2        2,565       105,933    172,348
--------------------------------------------------------------------------

Table: A case study. {#tbl:perf_metrics_case_study}


[^1]: pmu-tools - [https://github.com/andikleen/pmu-tools](https://github.com/andikleen/pmu-tools).