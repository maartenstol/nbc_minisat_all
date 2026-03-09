# Toda All-SAT Solver: Custom Instrumentation Modifications

This document outlines the specific modifications made to the original `nbc_minisat_all` source code. These changes were implemented to safely constrain the solver's execution volume and to extract per-solution trajectory metrics for downstream pipeline analysis.

## 1. Memory Structures (`solver.h`)
The original solver lacked mechanisms to track performance metrics over time or cap the maximum number of solutions internally. 

We introduced absolute snapshot tracking arrays directly into the `stats_t` struct. This avoids the computational overhead of calculating deltas during the inner search loop.
* Defined `MAX_TRACKED_SOLUTIONS 1000000` as a hard ceiling to prevent memory exhaustion.
* Added four `uint64*` arrays to track `decisions`, `propagations`, `inspects`, and `conflicts` at the exact moment each model is found.
* Added a `uint64 recorded_solutions` counter to bypass the GMP arbitrary-precision index complications.

## 2. Search Interruption (`solver.c`)
The original solver allowed unbounded model enumeration, relying on an external OS-level file size limit (`SIGXFSZ`) in the calling Python pipeline to abort runaway runs. This external kill signal was imprecise and risked data corruption.

We implemented a precise, internal graceful abort mechanism inside the `solver_search` function.
* Memory allocation (`malloc`) and deallocation (`free`) for the snapshot arrays were added to `solver_new` and `solver_delete`.
* Inside the model-found condition (`if (next == var_Undef)`), the solver now records the current global metrics into the snapshot arrays.
* When `recorded_solutions` reaches `MAX_TRACKED_SOLUTIONS`, the solver programmatically triggers `eflag = 1`. This mimics a manual user interrupt (`SIGINT`), forcing the solver to cleanly exit the search loop, print its final statistics, and terminate with a standard `0` exit code.

## 3. File I/O and CLI (`main.c`)
To extract the trajectory data without polluting the standard error stream or the standard output stream (which contains the actual boolean models), we added a dedicated file I/O pathway.

* **CLI Argument:** Added the `-s <filepath>` flag to the argument parser. 
* **Data Extraction:** Implemented the `dumpStatsCsv(stats* s_stats, const char* filepath)` function. This function is called at the end of execution to write the populated snapshot arrays to disk in a structured, comma-separated format.

```c
// Example invocation
./nbc_minisat_all_release input.cnf /dev/null -s trajectory_metrics.csv
```