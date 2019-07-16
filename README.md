# COSMA: Communication-Optimal, S-partition based, Matrix-Multiplication Algorithm

## Overview

COSMA is a parallel, high-performance, GPU-accelerated, matrix-matrix
mutliplication algorithm that is communication-optimal for all combinations of
matrix dimensions, number of processes and memory sizes, without the need of any
parameter tuning. The key idea behind COSMA is to first derive a tight optimal
sequential schedule and only then parallelize it, preserving I/O optimality
between processes. This stands in contrast with the 2D and 3D algorithms, which
fix process domain decomposition upfront and then map it to the matrix
dimensions, which may result in asymptotically more communication. The final
design of COSMA facilitates the overlap of computation and communication,
ensuring speedups and applicability of modern mechanisms such as RDMA. COSMA
allows to not utilize some processors in order to optimize the processor grid,
which reduces the communication volume even further and increases the
computation volume per processor.

COSMA alleviates the issues of current state-of-the-art algorithms, which can be
summarized as follows:

- `2D (SUMMA)`: Requires manual tuning and not communication-optimal in the
  presence of extra memory.
- `2.5D`: Optimal for `m=n`, but inefficient for `m << n` or `n << m` and for
  some numbers of processes `p`.
- `Recursive (CARMA)`: Asymptotically communication-optimal for all `m, n, k,
  p`, but splitting always the largest dimension might lead up to `√3` increase
  in communication volume. 
- `COSMA (this work)`: Strictly communication-optimal (not just asymptotically)
  for all `m, n, k, p` and memory sizes that yields the speedups by factor of up
    to 8.3x over the second-fastest algorithm.

In addition to being communication-optimal, this implementation is
higly-optimized to reduce the memory footprint in the following sense:

- `Buffer Reuse`: all the buffers are pre-allocated and carefully reused during
  execution, including the buffers necessary for the communication, which
  reduces the total memory usage.
- `Reduced Local Data Movement`: the assignment of data blocks to processes is
  fully adapted to communication pattern, which minimizes the need of local data
  reshuffling that arise after each communication step.

The library supports both one-sided and two-sided MPI communication backends. It
uses `dgemm` for the local computations, but also has a support for the `GPU`
acceleration through our `Tiled-MM` library using `cublas` 


## Building

The project uses submodules, to clone do :

```bash
git clone --recursive https://github.com/eth-cscs/COSMA.git
```

CMake is used to build the project. Required dependencies  are `MPI`, `MKL`,
`grid2grid`, `options` and `semiprof`. The last three dependencies can be
installed using `scripts/install_dependencies.py`. 

If the GPU backend is used, then `TiledMM` is required. 

For unit tests, a required dependency is `gtest_mpi` (a git submodule).

The following script provide information on important build variables and
options: `scripts/build.sh`. The script can be used as a template to build COSMA
for your system.


## Testing

In the build directory, do:
```bash
make test
```

## Miniapps

### Matrix Multiplication

The project contains a miniapp that produces two random matrices `A` and `B`,
computes their product `C` with the COSMA algorithm and outputs the time of the
multiplication.

The miniapp consists of an executable `./build/miniapp/cosma-miniapp` which can
be run with the following command line (assuming we are in the root folder of
the project):

>### Example:
>```
>n_iter=1 mpirun --oversubscribe -np 4 ./build/miniapp/cosma-miniapp -m 1000 -n 1000 -k 1000 -P 4 -s pm2,sm2,pk2
>```
>The flags have the following meaning:
>
>- `-m (--m_dimension)`: number of rows of matrices `A` and `C`
>
>- `-n (--n_dimension)`: number of columns of matrices `B` and `C`
>
>- `-k (--k_dimension)`: number of columns of matrix `A` and rows of matrix `B`
>
>- `-P (--processors)`: number of processors (i.e. ranks)
>
>- `-s (--steps, optional)`: string of triplets divided by comma defining the splitting strategy. Each triplet defines one step of the algorithm. The first character in the triplet defines whether it is a parallel (p) or a sequential (s) step. The second character defines the dimension that is splitted in this step. The third parameter is an integer which defines the divisor. This parameter can be omitted. In that case the default strategy will be used.
>
>- `-L (--memory, optional)`: memory limit, describes how many elements at most each rank can own. If not set, infinite memory will be assumed and the default strategy will only consist of parallel steps.
>
>- `-t (--topology, optional)`: if this flag is present, then ranks might be relabelled such that the ranks which communicate are physically closer to each other. This flag therefore determines whether the topology is communication-aware.

### Dry-run for statistics

If interested in the communication or computation volume, maximum buffer size or
a maximum local matrix-multiplication size, you can use the dry-run, which
simulates the algorithm without actually performing any communication or
computation. This dry-run mode is not distributed, even though it simulates a
distributed version of COSMA.

>### Example:
>The meaning of flags is the same as in previous examples. It can be run from the project directory with:
>```
>./build/miniapp/cosma-statistics -m 1000 -n 1000 -k 1000 -P 4 -s pm2,sm2,pk2
>```
>Executing the previous command produces the following output:
>
>```
>Matrix dimensions (m, n, k) = (1000, 1000, 1000)
>Number of processors: 4
>Divisions strategy:
>parallel (m / 2)
>sequential (m / 2)
>parallel (k / 2)
>
>Total communication units: 2000000
>Total computation units: 125000000
>Max buffer size: 500000
>Local m = 250
>Local n = 1000
>Local k = 500
>```

All the measurements are given in the units representing the number of elements
of the matrix (not in bytes).


## Profiling

Use `-DCOSMA_WITH_PROFILING=ON` to instrument the code. We use the profiler
called `semiprof`, written by Benjamin Cumming (https://github.com/bcumming).

### Example

Running the miniapp locally (from the project root folder) with the following
command:

```bash
mpirun --oversubscribe -np 4 ./build/miniapp/cosma-miniapp -m 1000 -n 1000 -k 1000 -P 4 -s pm2,sm2,pk2
```

Produces the following output from rank 0:

```
Matrix dimensions (m, n, k) = (1000, 1000, 1000)
Number of processors: 4
Divisions strategy:
parallel (m / 2)
sequential (m / 2)
parallel (k / 2)

_p_ REGION                     CALLS      THREAD        WALL       %
_p_ total                          -       0.110       0.110   100.0
_p_   multiply                     -       0.098       0.098    88.7
_p_     computation                2       0.052       0.052    47.1
_p_     communication              -       0.046       0.046    41.6
_p_       copy                     3       0.037       0.037    33.2
_p_       reduce                   3       0.009       0.009     8.3
_p_     layout                    18       0.000       0.000     0.0
_p_   preprocessing                3       0.012       0.012    11.3
```
The precentage is always relative to the first level above. All time
measurements are in seconds.

### Authors

- Grzegorz Kwasniewski, Marko Kabic, Maciej Besta, Raffaele Solca, Joost
  VandeVondele, Torsten Hoefler

### Questions?

For questions, feel free to contact us!
- For questions regarding theory, contact Grzegorz Kwasniewski (gkwasnie@inf.ethz.ch).
- For questions regarding the implementation, contact Marko Kabic (marko.kabic@cscs.ch).

### Ackowledgements

We thank Thibault Notargiacomo, Sam Yates, Benjamin Cumming and Simon Pintarelli
for their generous contribution to the project: great ideas, useful advices and 
fruitful discussions.

