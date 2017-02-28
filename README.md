# Notes---Polyhedral-Compilation

High Level Synthesis Tools for designing programs written in Behavioral Programming Language like C/C++.

Behavioral Language - Modular, composite program design. Consists of threads of program behavior inter-woven at runtime that together simulate the behavior of the program.

- Ideally, we would like to have a HLS that can write target optimized and specialized design in a Behavioral language like C/C++.  Attempts have been made but the designer still has to hand code a lot of parts. Key issues: On-chip buffer management, choice of degree of parallelism / pipelining, extent of prefetching, etc.
- Objective: Develop Automated compiler support for source to source transformations using advances in Polyhedral compilation.

Section 3.
In this section:
- Multi-stage process to efficiently transform a C program to run on FPGAs. 1. Method uses design space exploration to determine best performing program variant. 2. Search of best variants done through exploration of different rectangular tile sizes. 3. Each tile size gives a new program variant - in terms of - communication schedule, loop to be tiled etc. 
Each program variant is built as follows (Brief Overview)
- Use Polyhedral loop transformation. Objective: Maximize - Number of loops that can be tiled. Maximize - Data locality.
- Promote data accesses to on-chip buffers in the transformed program. Data reuse between consecutive iterations of the loop are exploited. Here, hardware buffer size constraint is preserved.
- Perform HLS specific coarse / fine grained optimizations.

Section 3.1
What is Polyhedral Compilation?
￼
 Polyhedral compilation is a way of modelling iterations of loop nests into points on a multidimensional space. Each dimension of the space represents one level of the loop nest and every point in the space refers to the state (or the value of the loop iterators) in one execution of the loop statement. Once this is done, we run a dependence analysis on the loop to check which instances of the loop statement execution depend on each other and hence may cause hindrance to parallelize the loop. For ex: Consider the following loop nest -  for (i = 1; i < N; i++) 	for (j = 1; j < M; j++) 		A[i][j] = SCALAR_VAL(0.5) * (A[i-1][j] + A[i][j-1]);  In this case, observe that since evaluation of A[i][j] depends on its neighbors A[i-1][j] & A[i][j-1], there is a dependence in the sense that we cannot evaluate the value of A[i][j] before we evaluate the value of A[i-1][j] & A[i][j-1] (they themselves are dependent on A[i-2][j] & A[i-1][j-1] and A[i-1][j-1] & A[i][j-2] and so on…). Hence, we get a dependence graph on the iteration kernel as shown in the above figure. 
The loops identified as having affine expressions are called SCoPs (Static Control Parts)

- It is a flexible and expressive framework used for loops that contain statically predictable control flow.
- Loop nests in which statements use affine accesses (Linear function of induction variable and program parameters) of array elements are called Static Control Parts.


Section 3.2
The next step is to perform polyhedral optimizations on the program. A single step in polyhedral compilation captures what corresponds to a sequence of several textbook loop optimizations. It carefully crafts a multidimensional affine schedule and as a result transforms the iteration space.
- Make polyhedral optimization that is geared towards maximizing coarse grained parallelism and data locality.
- Implemented through a complex combination of multidimensional tiling, skewing, fusion etc. Also known as the Tiling Hyperplanes method.
- If possible, parallel loops are brought to outer levels —> this technique is applied on every SCoP of the program.

Section 3.3
Automatic On-Chip buffer management.
- Most devices use a hardware-managed caches to store frequently accessed data.
- For FPGAs, the on-chip cache is much smaller than the data it tries operates on.
- There is a need for cache with explicit copy-in & copy-out instructions.
- Proposed method does the following: 1. Estimates the *On-chip buffer size*  and *associated communications* in one iteration of a loop  2. Data reuse between successive iterations of loop.
- When the above technique is applied to the  1. Innermost loop - It gives the number of registers required by the program. 2. Outermost loop - Equivalent to the entire program space.

How the technique is designed -
Use Parametric Polyhedral Sets to represent the set of Array elements used at a particular time in computation. These correspond to - Memory Accesses (Reuse / Write), Communication sets.
Change memory references in original code to on-chip buffer references.

Definitions:
1. Data space. All unique array elements accessed by a bunch of statements. (Polyhedra of array accesses)
2. Parametric Domain Slide (PDS) - The polyhedra represented by fixing bounds on all outer loop params. (Do not fix bounds on loop under consideration) - (Polyhedra of Iteration Domain)
3. Data space of a loop iteration - Polyhedra representing unique data elements accessed relative to iteration domain.
4. Reuse space - Array elements reused in successive iterations.
5. Per-Iteration communication - New array elements accessed in an iteration
6. Initialization - Data elements that should be in memory at the time the computation begins
7. Per-Iteration copy-out - Data locations that are going to be “written”

CodeGen-
Local Buffer Computation - Obtained by taking Rectangular Convex Hull of domain along every dimension.
CodeGen Algorithm -
1. createLocalBufferDecl —> Creates declaration to local array of same data type - Argument array. Size calculated by rectangular hull of argument set.
2. createScanningCodeIn —> Creates scan for all elements in Polyhedral set.
3. createScanningCodeOut  
4. converGlobalToLocalRef —> Converts all references to the global array with references to local in-cache array

HLS Specific Optimizations;
Fine grain parallelism exposure, Pipelining exposure.
1. Communication Prefetching and Overlapping. Evident from before, Prefetching data at iteration i = PerIterCommunication at iteration i+1.
2. Pipelining. Start execution of successive iterations of the loop - (Since fetch, compute algorithmic units are independent) Expose Inner loop parallelism —> Done by applying constraints like (#pragma AP pipeline II=1 ) while and after Tiling Hyperplanes method. (#pragma AP dependence inter false). There exists a large amount of parallelism inside a single loop iteration. This can be captured by polyhedral scheduling. Focusing on a single loop at a time by using the appropriate PDS —> compute a graph of dependent operations within a given iteration of the loop body.  This dependence graph is used to re- structure the code —> collection of separate functions for tasks that can be run in parallel

Section 4. Resource-Constrained On-Chip Buffer Management
Algo:
1. Build solution space.
2. Find optimal solution —> Satisfying memory and bandwidth constraints.

Solution space = For the cross product of arrays and loops —> Calculate ( buffer size, bandwidth requirement ) tuple for each combination.

Optimization Problem =  	Sum of buffer sizes < Available on-chip resources. such that. Bandwidth use is maximized.

How to get max bandwidth use? Choose effective tile sizes. TBD in Section 5.
