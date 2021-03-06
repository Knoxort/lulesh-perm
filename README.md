Checkpoint HPC Applications with PerMA Allocator
Scott Lloyd


Introduction
When moving to exascale systems, checkpoints to a parallel file system 
can challenge the network infrastructure with an excessive load and may 
possibly take longer than the mean time between failure (MTBF) for the 
system. A checkpoint method called Scalable Checkpoint and Restart (SCR) 
[1] has been previously developed to address this problem by using 
distributed storage such as node local flash, or RAM disk for high 
performance checkpoints at scale. This work introduces a checkpoint 
method complementary to SCR that identifies persistent data 
transparently for distribution within the SCR framework. Explicit code 
to write program variables and objects to a checkpoint file are not 
necessary. Objects identified as persistent are transparently saved at 
checkpoint time. The new checkpoint method uses a persistent memory 
allocator called PerMA that reduces the number of lines of code 
necessary to introduce checkpoint capability to an application. 

The PerMA memory allocator replaces the standard 'C' dynamic memory 
allocation functions with compatible versions that provide persistent 
memory to application programs. Memory allocated with the PerMA 
allocator will persist between program invocations after a call to a 
checkpoint function. This function essentially saves the state of the 
heap and registered global variables to a file which may reside in flash 
memory or other node local storage. A few other functions are also 
provided by the library to manage checkpoint files. Global variables in 
an application can be marked persistent and be included in a checkpoint 
by using a compiler attribute defined as PERM. The PerMA checkpoint 
method is not dependent on the programming model and works with 
distributed memory or shared memory programs. 

The current PerMA allocator, called perm-je, is implemented with 
jemalloc, an open source dynamic memory allocator [2], and with a module 
containing the additional management functions. A key feature of the 
persistence module is that integration with the allocation functions 
(malloc, free, etc) of various heap managers is simple. Very little 
change was required to incorporate jemalloc. The source for perm-je is 
available from:

https://computation.llnl.gov/casc/dcca-pub/dcca/Downloads_files/perm-je-latest.tar.gz

Two methods can be used to introduce persistent variables into an 
application, namely full persistence and partial persistence. The full 
persistence method makes all variables persistent while the partial 
persistence method identifies a core set of variables as persistent. 
Remaining global variables must be reinitialized upon restart. Achieving 
full persistence is simple to implement; however, reaching partial 
persistence requires an intimate knowledge of the application source 
code and data flow within the application and requires 
application-specific restart code. An example program with full 
persistence is shown in Figure 1. 

--------------------------------------------------------------------------------
Figure 1. Example checkpoint and restore program with full persistence.

/* 'C' program showing usage of persistent memory functions */
 
 #include <stdio.h>
 #include <string.h>
 #include "jemalloc/jemalloc.h"
 
 #define BACK_FILE "/tmp/app.back"
 #define MMAP_FILE "/tmp/app.mmap"
 #define MMAP_SIZE ((size_t)1 << 30)
 
 typedef struct {
     /* ... */
 } home_st;
 
 PERM home_st *home; /* use PERM to mark home as persistent */
 
 int main(int argc, char *argv[])
 {
     int do_restore = argc > 1 && strcmp("-r", argv[1]) == 0;
     char *mode = (do_restore) ? "r+" : "w+";
 
     /* call perm() and open() before malloc() */
     perm(PERM_START, PERM_SIZE);
     mopen(MMAP_FILE, mode, MMAP_SIZE);
     bopen(BACK_FILE, mode);
     if (do_restore) {
         restore();
     } else {
         home = (home_st *)malloc(sizeof(home_st));
         /* initialize home struct... */
         mflush(); backup();
     }
 
     for (;/* each step */;) {
         /* Application_Step(); */
         backup();
     }
 
     free(home);
     mclose();
     bclose();
     return(0);
 }
--------------------------------------------------------------------------------

LULESH
Three partial persistence versions of LULESH have been implemented. One 
is based on OpenMP (luleshOMP-perm) and two are based on MPI. Of the MPI 
versions, one is implemented without SCR (luleshMPI-perm) and one with 
SCR (luleshMPI-pscr). A minimal set of persistent variables were 
identified in LULESH through data-flow analysis assisted by a modified 
version of valgrind (see Figure 2). Variables used in one time step but 
defined in a prior step are identified as persistent. Read-only global 
variables are initialized at program startup. 

--------------------------------------------------------------------------------
Figure 2. LULESH Data-Flow Analysis. These variables are written to 
(defined) in the main loop. Those with a '*' are used as temporary 
variables and are initialized before use within a single iteration of 
the loop. Those without a '*' have loop carried flow (true) dependencies 
on variables from a prior loop iteration. Other variables in the program 
are used as local temporaries or are initialized at program start and 
are read-only thereafter. To preserve execution state, the minimal set 
of variables that must be checkpointed or made persistent are those 
without a '*'. 

Defined in main loop: * are temporary
                   e
                   p
                   q
                  ql *
                  qq *
                   v
                delv *
                vdov *
              arealg *
                  ss
                   x
                   y
                   z
                  xd
                  yd
                  zd
                 xdd *
                 ydd *
                 zdd *
                  fx *
                  fy *
                  fz *
                time
           deltatime
           dtcourant
             dthydro
               cycle
--------------------------------------------------------------------------------

When making variables persistent, care must be taken to make sure that 
persistent variables are not used to initialize static variables that 
are read-only after initialization. Otherwise, when restarted, the 
static variables will be based on a later state and not the initial 
state. 

Partial Persistence with PerMA. Table 1 shows the details from a run on 
8 Hyperion nodes that write checkpoint data to a local RAM file. The 
checkpoint file size for LULESH with partial persistence is 12 MB on 
each node. Even though the allocated persistent data totals 8.3 MB, a 
checkpoint file size of 12 MB is observed because of heap management 
overhead and rounding to the nearest allocation chunk size. In 
comparison, LULESH with full persistence requires a checkpoint file of 
at least 80 MB on each node. 

--------------------------------------------------------------------------------
Table 1. LULESH Run Summary
Detail                       Value
---------------------------- ---------
Iteration Step               212.9 msec
Checkpoint                   6.9 msec
Checkpoint overhead per step 3.2%
Checkpoint file size         12 MB (3 chunks in heap)
Allocated Persistent         8,318,008
Allocated Temporary          80-130 MB
--------------------------------------------------------------------------------

Partial Persistence with SCR. Working from the partial persistence 
version of LULESH-MPI, SCR was integrated with the PerMA checkpoint 
method. Integration is straightforward since SCR uses a similar 
file-based checkpoint model. The SCR routines are added around the PerMA 
backup() function. During a checkpoint, the persistent heap of each node 
is saved to the SCR file framework. Nodes query SCR for a new filename 
before each checkpoint. Figure 3 shows the output at the point of error 
and restart. The error condition is detected by SCR scripts and the 
restart is done automatically as another job step within the SLURM 
allocation. 

--------------------------------------------------------------------------------
Figure 3. Output of LULESH, SLURM, and SCR scripts at the point of error and restart.
...
0: cycle = 22, time = 4.197122e-06, dt=1.877104e-07
0: cycle = 23, time = 4.382200e-06, dt=1.850779e-07
srun: error: cab7: task 2: Aborted (core dumped)
srun: First task exited 30s ago
srun: tasks 0-1,3-7: running
srun: task 2: exited abnormally
srun: Terminating job step 580529.1
0: slurmd[cab4]: *** STEP 580529.1 KILLED AT 2013-01-02T11:23:25 WITH SIGNAL 9 ***
...
7: slurmd[cab26]: *** STEP 580529.1 KILLED AT 2013-01-02T11:23:25 WITH SIGNAL 9 ***
6: slurmd[cab23]: *** STEP 580529.1 KILLED AT 2013-01-02T11:23:25 WITH SIGNAL 9 ***
scr_retries_halt: CONTINUE RUN: No halt condition detected.
scr_retries_halt: CONTINUE RUN: No halt condition detected.
scr_srun: RUN 2: Wed Jan  2 11:24:26 PST 2013
0: restarting...
0: cycle = 24, time = 4.567278e-06, dt=1.850779e-07
0: cycle = 25, time = 4.752356e-06, dt=1.850779e-07
...
--------------------------------------------------------------------------------

References
1. A. Moody, G. Bronevetsky, K. Mohror, and B. R. de Supinski, "Design, modeling, and evaluation of a scalable multi-level checkpointing system," in Proceedings of the 2010 ACM/IEEE International Conference for High Performance Computing, Networking, Storage and Analysis, ser. SC ??10. Washington, DC, USA: IEEE Computer Society, 2010, pp. 1??11.
2. jemalloc, dynamic memory allocator, http://www.canonware.com/jemalloc/
3. Valgrind http://valgrind.org/
