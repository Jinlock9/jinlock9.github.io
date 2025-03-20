---
layout: post
title: Very Long Instruction Word (VLIW) Architecture 
date: 2025-03-19 22:00 +0800
categories: [Review, Computer_Architecture]
tags: computer_architecture vliw_architecture vliw
author: jinlock
description: Exploring VLIW Architecture
toc: false
published: true
---

I realized that I had completely misunderstood the fundamental concept of VLIW architecture while exploring superscalar architectures and parallel processing. I had mistakenly thought of VLIW as a type of superscalar processor that supports multiple functional units. However, I have now come to understand that VLIW processors rely entirely on the **compiler for instruction scheduling** and follow a **strict in-order execution model**. Given this realization, I have decided to carefully review VLIW architecture once again.

---

### **What is VLIW Architecture?**
VLIW (Very Long Instruction Word) is a **parallel processing architecture** that **statically schedules multiple operations** within a **single instruction word**, allowing **multiple functional units** to execute in parallel **in a single cycle**.

#### **Key Characteristics**
- **Fixed-width instruction words** containing multiple operations.
- **Compiler-driven parallelism** instead of hardware-based dynamic scheduling.
- **Simple hardware** (no out-of-order execution, no dynamic scheduling).
- **High instruction-level parallelism (ILP)** if the compiler optimally schedules instructions.

### **VLIW Instruction Packing & Execution**
VLIW processors require **wide instruction words** to encode multiple operations per cycle.

#### **Instruction Format**
A **typical VLIW instruction** consists of **multiple fields**, each assigned to a specific **functional unit**:
```
Example:

|     ALU Op     |     FPU Op     |  Load/Store Op   | Branch Op  |
|----------------|----------------|------------------|------------|
| ADD R1, R2, R3 | MUL R4, R5, R6 | LOAD R7, MEM[R8] | JUMP Label |
```
- The compiler ensures that **all operations in an instruction word can execute in parallel**.
- Functional units operate **independently** but execute **simultaneously**.

### **VLIW vs. Superscalar Processors**
1. **Instruction Scheduling**  
   - **VLIW Processor:** Static (done at compile-time)  
   - **Superscalar Processor:** Dynamic (done at runtime)  

2. **Execution Order**  
   - **VLIW Processor:** Fixed order – No reordering at runtime  
   - **Superscalar Processor:** Out-of-order execution possible  

3. **Branch Prediction**  
   - **VLIW Processor:** Not required  
   - **Superscalar Processor:** Required to avoid pipeline stalls  

4. **Hardware Complexity**  
   - **VLIW Processor:** Simpler (no dependency checking, no reorder buffer)  
   - **Superscalar Processor:** Complex (dynamic scheduling, speculation)  

5. **Compiler Role**  
   - **VLIW Processor:** Critical – Compiler handles ILP  
   - **Superscalar Processor:** Less critical – Hardware extracts ILP  

6. **Handling of Stalls**  
   - **VLIW Processor:** Entire instruction word stalls if one operation stalls  
   - **Superscalar Processor:** Independent instructions continue execution  

7. **Example Processors**  
   - **VLIW Processor:** Intel Itanium, TI C6000 DSP  
   - **Superscalar Processor:** Intel Core, AMD Ryzen    

---

### **Review & Discussion**

VLIW architecture was introduced as an alternative to traditional superscalar processors. By assigning all scheduling responsibilities to software, VLIW **removes hardware complexity**, such as intricate instruction scheduling and parallel dispatch [2]. Also, VLIW **enables fine-grained parallelism**, fully controlled by the compiler, whereas vector machines and multiprocessors provide coarse-grained parallelism, which is difficult for compilers to exploit [1].  

Because of this, the **compiler** is critical to the architecture. From the start, VLIW was designed with the assumption that it would be developed alongside its compiler [1]. Various software techniques, such as software pipelining and trace scheduling, exist to support VLIW [2].  

Since instruction scheduling is entirely handled by the compiler, it is crucial to extract enough ILP (Instruction-Level Parallelism) at compile time. Regular compilers typically work at the **basic block** level, but **basic blocks have severely limited parallelism**, making it necessary to develop compilers specifically optimized for VLIW [1]. The paper *"Parallel processing: a smart compiler and a dumb machine."* introduces **trace scheduling**, a technique that helps **increase ILP during machine scheduling** by treating multiple basic blocks as a single large basic block [1].

#### *"To avoid parallel limitation, increasing path length, excessive code motion, pipeline stalls because of branches, and many other problems which limit the performance of a VLIW processors, hardware and software scheduling techniques were proposed. [2]"*
This sentence is especially critical for me because I am currently working on Machine Code Sinking, a compiler-level optimization technique that performs code motion and introduces extra control paths (branches). This is unavoidable since I have to work with LLVM. I may need to discuss later how to customize an optimization technique designed for sequential programs to fit the VLIW architecture.

### **Reference**
- **[1]** Fisher, Joseph A., John R. Ellis, John C. Ruttenberg and Alexandru Nicolau. “Parallel processing: a smart compiler and a dumb machine.” SIGPLAN Conferences and Workshops (1984).  
- **[2]** Popescu, Cornel and Francisc Iacob. “MODEL SIMULATION AND PERFORMANCE EVALUATION FOR A VLIW ARCHITECTURE.” (2000).