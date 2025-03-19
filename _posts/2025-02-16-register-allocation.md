---
layout: post
title: Register Allocation
date: 2025-02-16 00:00 +0800
categories: [Review, Compiler]
tags: register_allocation compiler compiler_optimization
author: jinlock
description: Reviewing about Register Allocation
toc: false
published: true
---

As I explored the advantages of machine code sinking and the potential scenarios that could lead to performance degradation, I realized the importance of handling registers.

At school, I primarily learned about performance bottlenecks caused by accessing main memory due to cache misses. As a result, I had not given much thought to situations where the available register set is insufficient to hold all live values.

I encountered this problem when I attempted to allow code sinking under certain conditions. I initially assumed this would not worsen performance, but in some cases, it led to even greater performance degradation. This was due to high register pressure in the parent block.

I came across the concept that 
#### *"This secondary effect that results from instruction scheduling in large code segments is called register pressure." [1]*
As I consider merging basic blocks to improve instruction-level parallelism (ILP) for machine scheduling, I realize that understanding register management is crucial before working with large code segments. Efficient register handling will help mitigate register pressure and optimize performance.

---

### **Register Allocation**
**Register allocation** is the process of assigning program variables to a limited number of CPU registers to optimize execution speed and minimize memory access. Since registers are much faster than main memory, efficient allocation can significantly improve performance.

Key idea is that not every varaiable is **in-use ("live")**. Therefore, when there are not enough registers, some variables maybe move to and from RAM, and this is called ***spilling*** the register. And those varaiables which can be spilled and stored in register are considered as ***split***. And the high ***register pressure*** means there are more spills and reloads are needed [2].

Register allocation determines where variables are stored during execution — either in registers or memory. The allocator must decide **which registers to use** and **how long a variable should stay** in a specific location [2].

#### **Key Steps in Register Allocation**
- Liveness Analysis – Determines which variables are "live" at each point in the program, meaning they hold a value that will be used later.
- Register Assignment – Allocates registers to variables based on their liveness and usage frequency.
- Spilling – If there are more live variables than available registers, some variables are stored in memory and reloaded when needed.
- Move Optimization (Coalescing) – Reduces unnecessary register-to-register moves to improve efficiency.

#### **Common Challenges in Register Allocation**
- Aliasing: Some architectures allow different-sized accesses to the same register, causing unintended side effects.
- Pre-coloring: Some registers are reserved for specific functions (e.g., function arguments in calling conventions).
- NP-Completeness: Register allocation is computationally complex since it can be reduced to the graph coloring problem.

#### **Register Allocation Strategies**
- Graph Coloring Allocation – Represents variables as nodes in a graph, where edges indicate conflicting variable usage. Registers are assigned using graph coloring techniques.
- Linear Scan Allocation – A simpler, faster approach used in Just-In-Time (JIT) compilers, which processes variables in a linear order based on their lifetimes.
- Priority-based Allocation – Assigns registers based on variable usage frequency, favoring frequently used variables.

Now, let's dig into LLVM's primary strategies: **linear scan allocation** and **greedy register allocation**.

---

### **Linear Scan Allocation**
I referred to the lecture note by *Chaofan Lin*.

#### **Basic Definitions on Liveness**
1. A variable is **live** if it may be read in the later before it is written.
    - *"define"*: writing a varaible
    - *"use"*: reading a varaible
    - *"dead"*: old variable
2. The **live range** of a varaiable is the set of points at which that *varaiable is live*.
    - Ex. `[0, 2]`, `[4, 7]`, `[9, 15]`
3. The **live interval** of a variable is the smallest interval containing the *live range*.
    - Live interval **ignores holes** -> conservative estimate of the live range

#### **Basic Linear Scan Algorithm**
- **Linear Scan Register Allocation (LSRA)** proceeds the live intervals sorted by their *start positions* and allocate register greedily:
    - If a live interval **begins**, it selects a free register and allocate.
    - If a live interval **ends**, the allocated register is marked as *free* again.
    - If a live interval begins but **no free registers**, *spill* it.
- Implementation
    - maintains a **active** list: contains all intervals that overlap with the current positions and have registers assigned.
    - collects liveness information using *dataflow analysis algorithm*
    - conditions and loops -> not linear
        - use **CFG-based liveness analysis**
    - simple and runs very fast
    - *not good enough*

#### **Second Chance Binpacking**
As *LSRA* ignore holes in the live intervals, a **binpacking** model deals allocations in a finer grain size.

- **Binpacking Model**
    - **bin**: each register
    - pack variables' *lifetime* into bins
    - put two **non-overlapping** lifetime into the same bin
    - assign two variables in the same bin if one's lifetime is **entirely contained in a lifetime hole** of the other.
    - *free registers**: non-occupied registers + registers containing in lifetime holes
    - heuristically chooses the register with the **smallest hole** that is **larger** than the *variable's lifetime*.
- **Second Chance Allocation**
    - **spill another variable** to create a free register
    - compares the distance to each variable's **next reference** and weighted by the **the depth of the loop** it occurs.
        - Spills the one with lowest priority
    - split variables' *life intervals*
    - assigned new register when the spilled variable is needed again: give a *second chance*

---

### **Greedy Register Allocation**
*Greedy Register Allocation* is the register allocation algorithm introduced in LLVM 3.0 to improve the efficiency of register allocation while maintaining good performance.  
It replaces the earlier *linear scan* allocator and provides more flexible and extensible framework.

#### **Key Concepts**
1. **Spilling and Eviction-Based Strategy**
    - Instead of performing register allocation in a single pass, the allocator continuously refines its decisions, using heuristics to determin which values to keep in registers and which to spill to memory.
2. **Live Intervals and Interference Graph**
    - LLVM represents variable lifetimes using **live intervals**, which track the start and end of a value's usage in the program.
    - When a register is needed but unavailable, the allocator examines the **interference graph** (which shows conflicting live intervals) to decide the best strategy.
3. **Priority-Based Allocation**
    - Intervals are allocated based on a **priority queue**, where the priority is determined by:
        - Execution frequency (hot paths get higher priority).
        - Spill cost (how expensive it is to store/load values to/from memory).
4. **Spill Heuristics**
    - If no free register is available, the allocator considers spilling:
        - It prefers to spill varaibles that are used less frequently.
        - It tries to spill intervals that have minimal impact on execution performance.
5. **Splitting for Locality**
    - Instead of completely spilling a variable, **interval splitting** allows parts of the variable's lifetime to remain in registers while spilling other parts.
    - This reduces redundant spills and reloads, improving performance.
6. **Recoloring for Better Untilization**
    - If a register cannot be assigned due to conflicts, **live interval recoloring** attempts to adjust allocations to free up registers for critical values.

#### **Key Steps**
1. **Compute Live Intervals**:  
    Determine the lifetime of each value.
2. **Sort Intervals by Priority**:  
    Prioritize allocation based on execution frequency and spill cost.
3. **Try to Assign Registers**:  
    Allocate registers greedily, considering conflicts.
4. **Spill If Necessary**:  
    If no register is available, spill the least critical value.
5. **Split Intervals for Optimization**:  
    Reduce spill cost by splitting live intervals.
6. **Recolor If Needed**:  
    Reassign registers dynamically to optimize usage.

---

### **Reference**
- [1] J. L. Hennessy and D. A. Patterson, *Computer Architecture: A Quantitative Approach*, 6th ed. San Francisco, CA, USA: Morgan Kaufmann, 2019.  
- [2] "Register Allocation," Wikipedia, The Free Encyclopedia. [Online]. Available: https://en.wikipedia.org/wiki/Register_allocation. [Accessed: 16-02-2025].  
- [3] D. Greene, "Greedy Register Allocation in LLVM 3.0," LLVM Project Blog, Sept. 2011. [Online]. Available: https://blog.llvm.org/2011/09/greedy-register-allocation-in-llvm-30.html. [Accessed: 16-02-2025].