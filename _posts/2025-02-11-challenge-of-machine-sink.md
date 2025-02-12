---
layout: post
title: Challenges of Machine Code Sinking in VLIW Architectures - A Scheduling Perspective
date: 2025-02-11 02:22 +0800
categories: [Discussion, Compiler]
tags: vliw_architecture instruction_scheduling machine-sink
author: jinlock
description: Discussing performance degradation caused by LLVM machine-sink and possible solutions
toc: false
published: true
---

I am working on the development of an LLVM-based compiler for a custom Tensor Processing Unit. My supervisor's first directive was to analyze why LLVM's machine sinking adversely affects the number of cycles in executing certain kernels. 

We had been directly applying LLVM's built-in [Machine code sinking](https://llvm.org/doxygen/MachineSink_8cpp_source.html), but since an optimization technique that was expected to have a positive impact was instead causing a negative effect, a thorough investigation was necessary.

---

### What is Machine Code Sinking?
Machine code sinking is an optimization technique that moves instructions to later points in a program's execution where they are actually needed, reducing register pressure and potentially improving performance.  

By definition, machine code sinking is expected to improve performance, specifically execution time. It reduces register pressure, preventing potential register spills, and optimizes control flow to eliminate redundant computations.

---

### Approach 1

When I first started investigating this issue, I was new to both LLVM and compiler development, with only a basic understanding of VLIW architecture from college. As a result, I initially made some naive assumptions:  

#### The number of kernel cycles increases as the number of branches increases due to the splitting of critical edges in the program.
   
My reasoning was that the number of branches was the most apparent difference between the assembly output of the program with and without machine sinking. Additionally, our company’s branch prediction scheme was not well-suited for handling complex branching.  
   
However, after further analysis, I found that the difference in branch jumps between the two versions was not significant. As a result, I quickly discarded this assumption.

---

### Approach 2 

After that, I came up with a second assumption.  

#### Machine Code Sinking is Not Compatible with Pipeline Efficiency  

In some cases, I observed the following behavior:  

- **Before machine sinking:**  
    ```assembly
    LABEL1:
    r0 = Fsub r3 r4

    LABEL2:
    ...

    LABEL3:
    r2 = Fadd r0, r1
    ```  

- **After machine sinking:**  
    ```assembly
    LABEL1:

    LABEL2:
    ...

    LABEL3:
    r0 = Fsub r3 r4
    r2 = Fadd r0, r1
    ```  

This is a reasonable code movement from an optimization perspective. However, in actual cases, because these instructions have dependencies, the program with machine sinking incurred extra cycles due to pipeline stalls between the instructions. In contrast, the program without machine sinking was able to use precomputed data without stalling.  

I found this highly ironic. It seemed that code sinking introduced a cost (or penalty). Depending on the control path taken by the program, it could either **benefit** from skipping redundant computations or **suffer** from pipeline inefficiencies caused by sunk instructions.  

#### Refining the Sinking Decision 
To mitigate this issue, I introduced constraints to determine when to sink code:  
- **If the code is sunk along a split critical edge, and the "from" block's register pressure exceeds a predefined limit.**  
- **If the branch probability from the "parent block" to the successor block is fairly low.**  

My approach **only allowed sinking when it was more likely to have a positive effect**, as I believed that unnecessary code sinking was the root of the problem. 

However, this was not a true solution — it was more of a **loss reduction strategy**.  
Additionally, the results were highly biased by the test cases and types of kernels used, and in some cases, performance worsened even further.

Thus, this approach could never be a general solution—at best, fine-tuning the parameters could provide a temporary patch, but not a fundamental fix. This realization led me to reassess my assumptions: rather than machine sinking itself being flawed, the issue lay elsewhere.

While this approach mitigated some inefficiencies, it was clear that machine sinking itself wasn't fundamentally flawed. Instead of modifying the pass further, I shifted my focus to improving performance outside of machine sinking.

---

### Approach 3

At this point, I realized I had overlooked a crucial factor: 
#### instruction scheduling in VLIW architectures operates under fundamentally different constraints compared to traditional pipelined sequential processors.

For the same case with Part 1 Approach 2:

- **Before machine sinking:**  
    ```assembly
    LABEL1:
    r0 = Fsub r3 r4

    LABEL2:
    ...

    LABEL3:
    r2 = Fadd r0, r1
    ```  

- **After machine sinking:**  
    ```assembly
    LABEL1:

    LABEL2:
    ...

    LABEL3:
    r0 = Fsub r3 r4
    r2 = Fadd r0, r1
    ```  

This time, I analyzed how machine scheduling behaves in this scenario.
I realized that when machine scheduling was performed on Basic Block 3, there were only a small number of scalar instructions (including `Fsub` and `Fadd`). As a result, the scalar functional units were not fully utilized, and the instructions were not scheduled efficiently in the pipeline.

*Performing instruction scheduling separately for each basic block often leads to inefficient resource utilization, as most instructions leave many functional units underutilized* [1]. This is a common issue in VLIW architectures.

I thought that machine sinking can have a positive effect if it is complemented by better scheduling. If a global scheduling strategy can address local instruction shortages for certain resources, it can mitigate pipeline inefficiencies caused by injected code. As a result, machine sinking would contribute only positively to performance.
Therefore, I decided to find a way to make the MachineScheduler handle code scheduling from a global perspective. This could involve directly modifying it or introducing an intermediate pass between machine sinking and scheduling.

Moving forward, I will delve deeper into VLIW architecture and scheduling algorithms to refine this strategy. Additionally, I will assess whether this approach can provide a viable, generalizable solution beyond the test cases I have analyzed.

*I am open to any comments, feedback, or discussions regarding this topic. Feel free to share your thoughts!*

---

### Reference
[1] J. A. Fisher, "Trace Scheduling: A Technique for Global Microcode Compaction," IEEE Transactions on Computers, 1981.