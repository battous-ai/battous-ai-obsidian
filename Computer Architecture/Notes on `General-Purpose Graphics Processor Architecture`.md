# Chapter 1 Introduction
## 1.1 The Landscape of Computation Accelerators 
- improve performance → but the clock frequencies improves much more slowly
	- find more efficient hardware architectures
		- vector hardwares → 10x gain in efficiency by eliminating overheads of instruction processing
		- complex operations → by minimizing data movement
- yet flexible?
		- [[Turing Complete]]
## 1.2 GPU Hardware Basics
- Division of labor between CPU and GPU 
	- the beginning and ending of the computation require access to I/O devices
		- More API provides I/O services directly on the GPU
			- hide the complexity of managin communication between the CPU and GPU
			- but all assume the existence of a nearby CPU
- ![[image-20240128102815047.png|600]]
	- left (e.g. nvidia GPU)
		- discrete GPU setup
		- separate DRAM memory spaces for the CPU and GPU
			- CPU DRAM
				- low latency access
			- GPU DRAM
				- high throughput
	- right (e.g. AMD’s APU or mobile GPU)
		- integrated GPU
		- single DRAM memory
			- low power
		- private caches
			- ==[[cache-coherence problem]]==
- ==[[unified memory]]==
- GPU cores
	- _streaming multiporcessors_
		- the threads executing on a single core
			- communicate through a scratschpad memory
			- synchronize using fast barrier operations
		- first-level instruction
		- data caches
		- latency hiding to access memory ==by large numbers of threads==?
- high computation throughput
	- balance computational throughput with high memory bandwidth
		- → parallelism in the memory system
			- → include multiple memory channels in GPU
- improved performance on GPU
	- more area to arithmetic logic units
	- less area to control logic
- ![[image-20240128113032785.png|800]]
	- assume a simple cache model
		- no shared data between threads
		- infinite off-chip memory bandwidth
		- MC Region (CPU)
			- a large cache is shared among a small number of threads
				- threads &uarr;  performance &uarr;
		- Valley
			- cache cannot hold the entire working set → performance &darr;
		- MT Region (GPU)
			- multithreading hide long off-chip latency (tolerate frequent cache miss)
				- threads &uarr;  performance &uarr;
##  1.4 Book Outline
- Chapter 2: programming model
- Chapter 3: architecture of GPU core
- Chapter 4: memory system
- Chapter 5: additional research
# Chapter 2 Progamming Model
- exploit SIMD haredware
	- CUDA: MIMD-like programming model
	- at runtime, GPU hardware executes groups of scalar threads (called **warps**) in [[lockstep]] on SIMD hardware
		- the execution model is SIMT
## 2.1 Execution Model
- threads in a compute kernel ares organized into a hierarchg composed of a *grid of thread blocks* consisting of *warps*
- execute groups of thread together in lock-step → to improve efficiency
	- 32 threads → warp
	- warps → thread block or cooperative thread array (CTA)
- compiler and hardware enable the programmer to remain oblivious to the lock-step nature of thread execution in a warp
	- each thread appears to be independent
- threads within a CTA can communicate with each other via a per compute core scratchpad memory (**shared memory**)
	- each  streaming multiprocessor contains a single shared memory
		- the memory space is divided up among all CTAs running on that SM
		- as a software controlled cache
- threads *within* a CTA → synchronize using hardware-supported barrier instructions
- thread *in different* CTAs → synchronize through a global address space (accessible to all threads)
	- expensive
## 2.2 GPU Instruction Set Architectures
### 2.2.1 Nvidia GPU Instruction Set Architectures
- nvidia high-level virtual instruction set architecture → Parallel Thread Execution ISA (PTX)
# Chapter 3 The SIMT Core: Instruction and Register Data Flow
- GPU architecture
	- the SIMT cores that implement the computation
	- memory system
- ![[image-20240128175221982.png]]
	- the pipeline consists of 3 scheduling loops
		- instruction fetch loop
			- Fetch
			- I-Cache
			- Decode
			- I-Buffer
		- instruction issue loop
			- I-Buffer
			- Scoreboard
			- Issue
			- SIMT Stack
		- register access scheduling loop
			- Operand Collector
			- ALU
			- Memory
## 3.1 One-Loop Approximation
 - case of a single scheduler
	 - the unit of scheduling is warp
		 - each cycle the hardware selects one for scheduling
	 - the approximation
		 1. fetch the next instruction to execute for the warp 
			 - by the warp’s program counter 
				 - access a instruction memory
		 2. decode the instruction
		 3. 1) fetch the source operand registers
		 3. 2) determine the SIMT execution mask values (in parallel with fetching source operands from the register file) ?
			 1. ==how to determine? We can only predict?==
		 4. execution proceeds in a single-instruction, multiple-data manner
			- each thread executes on the function unit associated with a ==lane==? provided the SIMT execution mask is set
### 3.1.1 SIMT Execution Masking
- SIMT execution model
	- the abstraction that individual threads execute completely independently
	- achieved via a combination of
		- traditional prediciton
		- a tack of predicate masks (*SIMT stack*)
	- SIMT stack solves 2 issues
		- nested control flow 
		- ==skipping computation entirely while all threads in a warp avoid a control flow path==
- ==how does GPU hardware enable thread within a warp to follow different paths through the code while employing a SIMD datapath that allows only one instruction to execute per cycle?== (a.k.a branch divergence handling mechanism)
	- to serialize execution of threads following different paths within a given warp
	- to achieve this serialization of divergent code paths
		- stack with three entries
			- a reconvergence program counter (RPC)
				- where threads that diverge can be forced to continue executing in lock-step
			- the address of the next instruction to execute (Next PC)
			- an active mask
		- order of the stack entries following a divergent branch
			- entry with the most active threads first
			- then the entry with fewer active threads
### ==3.1.2 SIMT Deadlock and Stackless SIMT Architectures== *(hard!)*
- stack-based implementation of SIMT can lead to a deadlock condition
	- ![[image-20240130105258105.png]]
	- to reach the reconvergence point, all divergent threads must complete computation
		- but the other threads are still waiting for the mutex to release
			- which can only be completed when the reconvergence point is reached
				- hence **deadlock**
	- <u>so the threads must be somehow split again</u>
- nvidia new thread divergence management approach since Volta
	- called Independent Thread Scheduling
	- stackless
	- the key idea is to replace the stack with per warp convergence barriers
		- ~~so only ONE barrier at a time for a given warp?~~
	- ![[image-20240130120132208.png]]
	- stored in the register and used by hardware warp scheduler
	- *Barrier Participation Mask* 
		- think of it as a ticket for a thread to be allowed to participate in a certain convergence barrier
			- tracks which threads within a given warp participate in a given convergence barrier
			- used by the warp scheduler to stop threads (at a specific convergence barrier location which can be the immediate postdominator of the branch or another location)
			- threads tracked by a given barrier participation mask will wait for each other to reach a common point in the program following a divergent branch and thereby reconverge together
		- may be **more than one** barrier participation mask for a given warp
			- to support nested control flow
		- e.g. if a warp has 32 threads, then the mask is 32-bit wide
			- if a bit is set, the corresponding thread (in the warp) participates in this convergence barrier
	- *Barrier State*
		- tracks which threads have arrived at a given convergence barrier
			- <u>so different barrier state for different convergence biarrier?</u>
	- *Thread State*
		- tracks (the threads in a given warp)
			- is ready to execute
			- blocked at a convergence barrier 
				- if so, which one
			- has yielded
				- may be used to enable other threads in the warp to make forward progess past the convergence barrier in a situation that would otherwise lead to SIMT deadlock
	- *Thread rPC Field*
		- for each thread that is not active, the address of the nex instruction to execute
	- *Thread Active Field*
		- a bit indicates if the corresponding thread in the warp is active
		- <u>so only the threads which are active can be scheduled?</u>
	- detailed implementation
		- barrier participation mask initialization
			- a special “ADD” instruction is employed → active threads have their bit set in the convergence barrier indicated by the ADD instruction
		- branch → threads diverge 
			- select a subset of threads (called a warp split") with a common PC by the scheduler
			- update the ==Thread Active Field== to enable execution for these threads of the warp
			- teh scheduler is free to switch between groups of diverged threads
		- “WAIT” instruction
			-  to stop a warp split when it reaches a convergence barrier
			- The effect of the WAIT instruction
				- to add the threads in the warp split to the Barrier State register for the barrier
				- change the threads’ state to blocked
				- Once all threads in the barrier participation mask have executed the corresponding WAIT instruction the thread scheduler can switch all the threads from the original warp split to active and SIMD efficiency is maintained.
		- To enable switching between warp splits NVIDIA describes using a YIELD instruction along with other details such as support for indirect branches that we omit in this discussion
### 3.1.3 Warp Scheduling
-  One property of this scheduling order is that it allows roughly equal time to each issued instruction to complete execution. If the number of warps in a core multiplied by the issue time of each warp exceeds the memory latency the execution units in the core will always remain busy. So, increasing the number of warps up to this point can in principle increase throughput per core.
- However, there is an important trade off: to enable a different warp to issue an instruction each cycle it is necessary that each thread have its own registers (this avoids the need to copy and restore register state between registers and memory). Thus, increasing the number of warps per core increases the fraction of chip area devoted to register file storage relative to the fraction dedicated to execution units. For a fixed chip area increasing warps per core will decrease the total number of cores per chip.
- locality properties can either favor or discourage round-robin scheduling: when different threads share data at a similar point in their execution, such as when accessing texture maps in graphics pixel shaders, it is beneficial for threads to make equal progress as this can increase the number of memory references which “hit” in on-chip caches, which is encouraged by roundrobin scheduling [Lindholm et al., 2015]. Similarly, accessing DRAM is more efficient when nearby locations in the address space are accessed nearby in time and this is also encouraged by round-robin scheduling [Narasiman et al., 2011]. On the other hand, when threads mostly access disjoint data, as tends to occur with more complex data structures, it can be beneficial for a given thread to be scheduled repeatedly so as to maximize locality
## 3.2 Two-Loop Approximation
- to hide long execution latencies → increase number of warps per core
	- to reduce number of warps per core → to be able to issue a subsequent instruction from a warp while earlier instructions not yet completed
		- but the problem with one-loop approximation is
			- the scheduling logic **only** has access to 
				- the thread identifier
				- the address of the next instruction (to issue)
			- the scheduling logic has **no** access about
				- dependency information among instructions (e.g. the next instruction depends on the earlier unfinished instruction)
					- the necessary changes
						- first fetch the instruction from memory to an instruction buffer
							- where to place instructions after cache access
						- a separate scheduler to decide which instructions in the instruction buffer to issue next
- instruction memory
	- as first-level instruction cache
	- instruction information is placed into the instruction buffer after
		- a cache hit
		- a fill from a cache miss
- detect data dependencies between instruction within the same warp
	- the CPU ways
		- a scoreboard
			- in-order execution
				- for a single-threaded in-order CPU
					- each register is represented as a bit in the scoreboard
						- set whenever an instruction write to that register
						- clear by the instruction writing to the register
						- other instructions tat want to read or write to the register wait until the bit is cleared
			- out-of-order execution
		- reservation stations
- GPU implements in-order scoreboards
	- challenges when supporting multiple warps
	- problems
		- large number of registers → too many bits to implememt the scoreboard