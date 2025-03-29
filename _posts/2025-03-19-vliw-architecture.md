---
layout: post
title: Very Long Instruction Word (VLIW) Processor
date: 2025-03-19 22:00 +0800
categories: [Review, Computer_Architecture]
tags: computer_architecture vliw compiler
author: jinlock
description: Exploring VLIW Processor
toc: false
published: true
---

I mainly learned about hardware-based parallel execution and hazard detection in university, so exploring VLIW architecture was really interesting but also quite confusing. That’s why I want to look into it more deeply.

### Background

In computer architecture, **issuing an instruction** refers to the act of sending an instruction to a functional unit for execution. To reduce the cycles per instruction (CPI) to less than one, it is necessary to issue multiple instructions per clock cycle, since issuing only one instruction per cycle cannot achieve CPI lower than one. The solution is **multiple issue**.

Statically scheduled superscalar processors execute instructions in-order, while dynamically scheduled superscalar processors support out-of-order execution. Both architectures can issue a varying number of instructions per clock cycle depending on data dependencies and available resources.

---

### VLIW Processor

A **VLIW (Very Long Instruction Word) processor** issues a fixed number of instructions per clock cycle. These instructions are formatted either as a large instruction word or as a fixed instruction packet, with the parallelism among instructions indicated by the instruction itself [3]. VLIW architectures use multiple, independent functional units [3].

A typical VLIW instruction consists of multiple operation fields, each assigned to a specific functional unit. For example:

```
|     ALU Op     |     FPU Op     |  Load/Store Op   | Branch Op  |
|----------------|----------------|------------------|------------|
| ADD R1, R2, R3 | MUL R4, R5, R6 | LOAD R7, MEM[R8] | JUMP Label |
```

The compiler ensures that all operations within a VLIW instruction can execute in parallel. Functional units operate independently and execute simultaneously.

---

### "a smart compiler and a dumb machine"

VLIW processors are a *logical extension of superscalar RISC processors* [2], and were introduced as an alternative to traditional superscalar architectures. They exploit instruction-level parallelism using multiple issue and static scheduling [3]. Since VLIW processors rely on compiler-based static scheduling, they remove the need for hardware mechanisms such as hazard detection and instruction scheduling [2]. This results in simpler hardware, with all scheduling responsibilities handled by the **compiler**.

This approach presents challenges to the compiler. To fully utilize all functional units, there must be sufficient parallelism in the code. When using local scheduling, where instruction-level parallelism is extracted within a basic block, the available parallelism is often limited [1]. To address this, global scheduling techniques such as **trace scheduling** are used to find parallelism across multiple basic blocks [1].

---

### Discussion

> To avoid parallel limitation, increasing path length, excessive code motion, pipeline stalls because of branches, and many other problems which limit the performance of a VLIW processors, hardware and software scheduling techniques were proposed. [2]

This sentence is especially critical for me because I am currently working on Machine Code Sinking, a compiler-level optimization technique that performs code motion and introduces extra control paths (branches). This is unavoidable since I have to work with LLVM. I may need to discuss later how to customize an optimization technique designed for sequential programs to fit the VLIW architecture.

---

### **Reference**
- **[1]** Fisher, Joseph A., John R. Ellis, John C. Ruttenberg and Alexandru Nicolau. “Parallel processing: a smart compiler and a dumb machine.” SIGPLAN Conferences and Workshops (1984).  
- **[2]** Popescu, Cornel and Francisc Iacob. “MODEL SIMULATION AND PERFORMANCE EVALUATION FOR A VLIW ARCHITECTURE.” (2000).
- **[3]** John L. Hennessy and David A. Patterson. 2017. Computer Architecture, Sixth Edition: A Quantitative Approach (6th. ed.). Morgan Kaufmann Publishers Inc., San Francisco, CA, USA.