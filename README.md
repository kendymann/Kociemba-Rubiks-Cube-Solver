# A Two-Phase Rubik's Cube Solver with IDA* and Pruning Tables

**Authors:** JJ Truong & Brendan Greening  

## Abstract
We present a Rubik's Cube solver built in Java that uses Kociemba's two-phase algorithm, iterative deepening A* (IDA*), and precomputed pruning tables. Our work focuses on practical performance: we transition from early breadth-first and depth-first prototypes to a two-phase approach that reduces the search space from the full cube group to a restricted subgroup. We implement facelet, cubie, and coordinate representations to support fast move application and heuristic lookup. To meet runtime limits, we replace object-based node expansion with a structure-of-arrays manual stack to avoid garbage collection overhead. The final solver reliably completes the full test suite of 40 scrambles within the provided constraints.

## Introduction
The Rubik's Cube has approximately $4.3 \times 10^{19}$ reachable states, making naive graph search infeasible. Our objective was to build a solver that is correct, fast enough for automated testing, and implemented from first principles suitable for a data structures and algorithms course. We adopt Kociemba's two-phase framework, which uses group-theoretic structure to reduce the effective search space. We then engineer efficient representations and heuristics to make IDA* practical at scale.

## Algorithm Overview
Kociemba's algorithm splits the problem into two searches:
* **Phase 1 (G0 $\rightarrow$ G1):** The goal is to reach a state with correct edge and corner orientation and with the middle slice edges positioned in the middle layer. The allowed move set is unrestricted.
* **Phase 2 (G1 $\rightarrow$ Solved):** The state is solved using only moves that preserve the phase-1 constraints: $U, D, R2, L2, F2, B2$.

This reduces the search complexity by forcing phase 1 into a subgroup where phase 2 can be solved with a constrained move set.

## Cube Representations
We use three representations to support different computational needs:
* **Facelet cube:** A 54-sticker representation used for parsing and output.
* **Cubie cube:** A piece-based model that tracks the permutation and orientation of corners and edges. This is the core representation for move application.
* **Coordinate cube:** The cubie cube converted into integer coordinates for fast indexing into pruning tables. We store three coordinates: twist (0–2186), flip (0–2047), and slice (0–494).

## Heuristics and Pruning Tables
Our heuristics are perfect lower bounds derived from pruning tables:
1. Initialize the solved state's index with distance 0.
2. Apply all 18 moves to expand reachable indices; mark distance 1.
3. Continue breadth-first expansion until all table entries are filled.

During search, the distance value for the current coordinate is a lower bound on moves to reach the goal state. This guides IDA* while preserving admissibility.

## Search and Memory Optimization
Our initial IDA* used recursive node expansion with object allocation for every state. At depth, millions of objects triggered frequent garbage collection and caused timeouts. We replaced this with a **structure-of-arrays** approach:
* Parallel integer arrays (e.g., `moveStack`, `flipStack`, `twistStack`) store per-depth state.
* The depth index acts as a manual stack pointer, enabling constant-time backtracking.
* No node objects are created during search, keeping data hot in cache and eliminating GC pauses.

We considered nibble-packing for pruning tables, but standard byte arrays were small enough (on the order of 5–10MB total) to remain practical while avoiding bit-level complexity.

## Evaluation
We evaluated correctness by verifying that every produced solution transforms the input scramble into the solved state. After the structure-of-arrays rewrite and pruning table integration, the solver completed all 40 provided test cases within the time limit. Earlier approaches (BFS/DFS, then A* with weak heuristics) either exhausted memory or timed out.

## Discussion
The project highlights the gap between theoretical search guarantees and practical performance. The two-phase reduction was essential, but the decisive improvement came from engineering: data layout and allocation behavior dominated runtime for large searches. This experience reinforced the importance of algorithmic design paired with systems-aware implementation details.

## Conclusion
We built a Rubik's Cube solver that combines Kociemba's two-phase algorithm, IDA*, and pruning tables to reliably solve provided test cases under tight constraints. The implementation emphasizes efficient state representation and memory behavior, demonstrating that high-level algorithms and low-level data layout must align to achieve performance in real-world search problems.

## References
1. H. Kociemba. *The Two-Phase Algorithm*. http://kociemba.org/cube.htm
2. H. Kociemba. *The Mathematics of the Two-Phase Algorithm*. https://kociemba.org/math/twophase.htm
3. M. A. Khan. *Kociemba's Two-Phase Algorithm: How it Works and its Applications*. Medium. https://medium.com/@muhammadalikhan0003/kociembas-two-phase-algorithm-how-it-works-and-its-applications-3d8f97a3562a
4. M. Botta. *Rubik's Cube Solver Algorithms*. Journal of Physics: Conference Series. https://iopscience.iop.org/article/10.1088/1742-6596/2386/1/012018/pdf
5. *Bachelor Thesis on Kociemba Implementation*. https://www.diva-portal.org/smash/get/diva2:816583/FULLTEXT01.pdf
6. MIT OpenCourseWare. *The Mathematics of the Rubik's Cube*. https://web.mit.edu/sp.268/www/rubik.pdf
7. W. Konieczny. *Group Theory Applications in the Rubik's Cube*. https://www.fuw.edu.pl/~konieczn/RubikCube.pd
8. *Learning How to Solve a Rubik's Cube*. https://www.youtube.com/watch?v=PW2J8IblczM
