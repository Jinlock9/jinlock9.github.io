---
layout: post
title: Review - Software Pipelining - An Effective Scheduling Technique for VLIW Machines
date: 2025-04-01 23:00 +0800
categories: [Paper, Instruction_Scheduling]
tags: compiler instruction_level_parallelism instruction_scheduling software_pipelining
author: jinlock
description: Reviewing "Software Pipelining - An Effective Scheduling Technique for VLIW Machines"
toc: false
published: true
---

As I continue exploring VLIW processors, I have observed that exposing a sufficient level of Instruction Level Parallelism (ILP) is important to make effective use of their architectural characteristics. Most of the papers and textbooks I have consulted introduce trace scheduling as a primary compilation technique for uncovering ILP, which I plan to examine in more detail.

However, recently, I read a short retrospective article titled Retrospective: Software Pipelining – An Effective Scheduling Technique for VLIW Machines. In this piece, the author revisits her earlier work and argues that software pipelining can be effective on VLIW processors even without complex hardware mechanisms. To better understand this alternative scheduling approach, I decided to review the original paper, Software Pipelining: An Effective Scheduling Technique for VLIW Machines.

![Software Pipelining](../assets/img/posts/2025-04-01-paper-review-software-pipelining-for-vliw.png)

---
## The Basics

But before that, it is necessary to review the fundamentals of software pipelining in order to better understand the paper.

### **Software Pipelining**

**Software pipelining** is a loop scheduling technique that increases instruction-level parallelism by reorganizing loop execution such that each iteration of the transformed loop contains instructions drawn from multiple iterations of the original loop. By interleaving operations from different iterations, software pipelining separates dependent computations across a wider range of cycles—often by the length of an entire loop body—which reduces data hazards and increases the likelihood of producing a schedule free of stalls. This overlapping structure leads to more efficient use of hardware resources and significantly improves throughput. [3, 4]

#### Example [2]
- Before Software Pipelining (Original Loop):

  ```assembly
  Loop:   L.D     F0, 0(R1)      ; Load A[i]
          ADD.D   F4, F0, F2     ; A[i] + const
          S.D     F4, 0(R1)      ; Store back
          DADDUI  R1, R1, #-8    ; Move to A[i-1]
          BNE     R1, R2, Loop   ; Loop
  ```

- After Software Pipelining 

  ```assembly
  Loop:   S.D     F4, 16(R1)     ; Store A[i-2]
          ADD.D   F4, F0, F2     ; Add A[i-1] + const
          L.D     F0, 0(R1)      ; Load A[i]
          DADDUI  R1, R1, #-8    ; Move to A[i-1]
          BNE     R1, R2, Loop   ; Loop
  ```

- Each instruction comes from a different iteration (i, i-1, i-2).
- Overlapping iterations keeps the pipeline busy every cycle.
- Achieves one result per loop iteration → maximizes throughput.

---

### **Modulo Scheduling**

The commonly used technique to implement Software Pipelining is **Modulo Scheduling** [4].

The objective of modulo scheduling is to:
> **Assign a schedule to each instruction in a loop such that iterations can execute in an overlapped fashion—repeating every _Initiation Interval (II)_ cycles—without violating dependencies or exceeding resource limits.**

#### **Initiation Interval (II)**

- **II** is the number of cycles between the start of two consecutive loop iterations.
- A smaller II means more overlap and better throughput.
- Finding the **smallest possible II** that respects all constraints is the main goal.

#### **Scheduling Constraints**

1. Dependency Constraint
  - Instructions that depend on each other across loop iterations must be properly ordered.  
  For example, if instruction `A` produces a value needed by instruction `B` in a future iteration, we must ensure:
  - *Start(B) ≥ Start(A) + latency - distance × II*
  - **latency**: cycles it takes for A’s result to be ready  
  - **distance**: how many iterations ahead B is from A  
  - This ensures B never executes before A finishes.

2. Resource Constraint
  - At any cycle within the repeating loop of II cycles, resource usage must not exceed hardware limits.
  - For example:
    - If your hardware has only **one multiplier**, you can't schedule **two multiply instructions** in the same modulo II cycle.
  - This means:
    - For each type of hardware (ALU, multiplier, etc.), we must ensure no oversubscription.

#### **Scheduling Strategy**

1. Start with a small II (e.g., 1).
2. Try assigning cycle numbers to all instructions that:
   - Respect the dependency rule.
   - Stay within the hardware resource limits.
3. If no legal schedule is found, **increase II** and repeat.
4. When both constraints are satisfied, you've found a valid modulo schedule.

#### **Final Output**

Once a valid II is found:
- A **kernel** is constructed: a compact repeating sequence of scheduled instructions.
- This kernel is wrapped with:
  - **Prologue**: the pipeline warm-up phase.
  - **Epilogue**: the pipeline wind-down phase.
- The kernel is executed repeatedly, with new iterations starting every II cycles.

---

### **Software Pipelining vs. Loop Unrolling [3]**

**Software pipelining** can be viewed as a symbolic form of **loop unrolling** [3]. Some software pipelining algorithms even rely on unrolling techniques to help build efficient schedules.
Although both techniques aim to improve loop performance, they reduce different types of overhead.

#### **Loop Unrolling**

Loop unrolling reduces the overhead of loop control. This includes the time spent updating the loop counter and performing branch checks. By executing multiple iterations in one unrolled block, the number of times this overhead is paid is reduced.
For example, if a loop runs 100 times and is unrolled by a factor of 4, the loop control overhead is paid only 25 times instead of 100.
However, loop unrolling increases code size because it duplicates the loop body for each unroll.

#### **Software Pipelining**

Software pipelining improves performance by overlapping instructions from different iterations. It allows the loop to run closer to peak speed by avoiding idle cycles in the pipeline.
Instead of duplicating the loop body, software pipelining creates a compact kernel that repeats every few cycles. The overhead of filling and draining the pipeline is only paid once—at the beginning and end of the loop.
This makes software pipelining more space-efficient than loop unrolling.

#### **Why Use Both?**

Since loop unrolling and software pipelining target different types of inefficiencies, using them together can provide better results than using either one alone.
- Loop unrolling reduces how often loop control code runs.
- Software pipelining keeps the pipeline busy and avoids stalls.
When combined, these techniques can significantly improve the performance of inner loops, especially in performance-critical applications.

---

### Difficulties in Practice [3]

In practice, compiling loops using software pipelining presents several challenges:

1. **Loop Restructuring**: Many loops require substantial transformation before software pipelining can be applied.

2. **Trade-offs**: The balance between the overhead introduced by software pipelining and the performance gains it provides is often complex and context-dependent.

3. **Register Management**: Efficiently managing registers adds another layer of difficulty to the compilation process.

To address the latter two issues, the IA-64 architecture introduced extensive hardware support for software pipelining. While this hardware improves the efficiency of applying the technique, it does not eliminate the need for sophisticated compiler support or the need to make difficult decisions about how to best compile a loop.

---

## Paper Review [2]

The paper introduces **software pipelining** as an effective and practical scheduling technique for VLIW (Very Long Instruction Word) processors. Unlike traditional methods like trace scheduling, which optimize across basic blocks, software pipelining overlaps successive loop iterations at fixed initiation intervals. This enables multiple iterations to execute concurrently, achieving high throughput while keeping the object code compact.

Because finding an optimal schedule is **NP-complete**, earlier approaches either depended on specialized hardware (such as the crossbar in polycyclic architectures) or limited software pipelining to simple loops using heuristics.

This paper makes two major contributions:
1. It improves software scheduling heuristics and introduces **modulo variable expansion**, enabling near-optimal performance for loops with both intra- and inter-iteration dependencies—**without requiring specialized hardware**.
2. It proposes a **hierarchical reduction** technique, which allows complex control constructs (such as conditionals) to be treated like simple operations. This generalization makes it possible to apply software pipelining to **all innermost loops**, including those with conditional statements and loops with small iteration counts.

While **software pipelining** is the main reason for the speedup observed in programs, it is **hierarchical reduction** that makes it possible to attain **consistently good results**—even on programs that contain conditional branches in innermost loops or have very few iterations. Together, these two techniques allow the compiler to generate near-optimal, and sometimes optimal, code for VLIW machines.

The effectiveness of these methods is demonstrated in a compiler targeting the **Warp** VLIW architecture, with strong performance shown across a wide range of real-world applications in image, signal, and scientific processing.

---

### **Simple Loop**

In software pipelining, multiple iterations of a loop are overlapped by initiating a new iteration every cycle, even before previous iterations finish. This leads to better utilization of the processor's parallel resources.

- The loop execution is divided into three phases:
  1. **Prolog** – where the pipeline is filled by initiating new iterations.
  2. **Steady state** – where a constant number of iterations are in flight, achieving maximum throughput.
  3. **Epilog** – where the remaining iterations complete.

- The **goal** is to minimize the **initiation interval (II)**—the number of cycles between the start of consecutive iterations. A smaller II means higher throughput.

- The software pipelining problem is modeled as a scheduling problem under two main constraints:
  1. **Resource constraints**: All operations scheduled in the same cycle across different iterations must not exceed the hardware's available resources.
  2. **Precedence constraints**: Data dependencies must be respected, including inter-iteration dependencies (i.e., when an operation depends on a value from a previous iteration).

  - These constraints are expressed using a **modulo reservation table** and a **precedence graph**, where operations are represented as nodes with dependency edges.

#### Modulo Variable Expansion 
To further reduce II and improve scheduling flexibility, modulo variable expansion is applied. This optimization assigns different registers to the same logical variable across iterations, effectively breaking inter-iteration false dependencies caused by register reuse. By doing so, multiple overlapping uses of a variable do not block pipeline progress, enabling tighter packing of operations and more aggressive overlapping of iterations.

While this transformation can increase register pressure or require code unrolling, it significantly lowers the initiation interval in practice and is particularly well-suited for architectures with abundant registers, like VLIW machines.

#### Objective
The scheduling objective is to assign each operation a start time such that the same schedule can be reused across iterations, with the shortest possible constant initiation interval.

#### Note
> ... when register allocation becomes a problem, software pipelining is not as crucial (323).  

> Thus, we can conclude that the increase in code size due to software pipelining is not an issue (323).  

The above quotes suggest that software pipelining does not need to take register allocation or code size into serious consideration. This is particularly attractive, as code size is commonly cited as a trade-off in loop unrolling [3].

---

### **Hierarchical reduction**

**Hierarchical reduction** is a method that allows software pipelining to be applied to all innermost loops, even those containing conditional statements or with short iteration counts.

- The idea is to represent entire control constructs (like conditionals and loops) as single scheduling units, or **nodes**. This enables existing scheduling techniques (such as software pipelining) to be applied beyond basic blocks.

- For **conditional statements**, each branch (THEN/ELSE) is scheduled independently. The entire construct is then reduced to a single node whose constraints are the union of both branches. This allows operations outside the conditional to be scheduled in parallel with it.

- For **loops**, the prolog and epilog can be overlapped with operations outside the loop by reducing the loop to a single node, while the steady state is kept isolated to preserve correctness.

- Once control constructs are reduced to straight-line forms, global code motion techniques can be applied across them. This makes it possible to compact complex loops effectively and improve performance even in short or irregular loops.

The key benefit of hierarchical reduction is that *it generalizes software pipelining to support more complex control structures while maintaining efficiency and code compactness*.

#### Example
```c
for (int i = 0; i < N; ++i) {
  if (cond[i])
    A[i] = B[i] + 1;
  else
    A[i] = B[i] - 1;
}
```
Without hierarchical reduction:
- The compiler might treat this as two separate basic blocks and pipeline only each separately.

With hierarchical reduction:
- The if-else is reduced into a single operation.
- Software pipelining is applied to the entire loop structure as if the conditional is just one step in the pipeline.

---

### **Software Pipelining vs Trace Scheduling**
This is the part I'm really curious about: does software pipelining have an advantage over trace scheduling?

**Trace scheduling** is a global scheduling technique that optimizes the most frequently executed paths (called traces) by treating them like large basic blocks. It aggressively moves instructions across branches to increase parallelism. In contrast, **software pipelining** focuses on overlapping loop iterations to achieve high throughput. The key difference is that software pipelining retains the program’s control structure, while trace scheduling flattens it for optimization, often at the cost of increased code size.

#### Loops
Trace scheduling optimizes loop bodies by unrolling them, creating longer traces to expose more parallelism. However, this requires guessing how many iterations to unroll, and the overhead of filling and draining the pipeline prevents it from reaching optimal performance. Software pipelining, on the other hand, overlaps iterations directly and can achieve optimal throughput without trial-and-error tuning. It also avoids excessive code size by only unrolling when necessary (e.g., for modulo variable expansion).

#### Conditional Statements
With conditionals, trace scheduling assumes one path is more likely and optimizes that trace. This can lead to code duplication when preserving correctness for less-likely paths, causing code explosion. In contrast, software pipelining uses hierarchical reduction to treat an entire conditional as a single schedulable unit. This approach avoids duplication and allows pipelining even when the control flow is complex or data-dependent.

#### Conclusion
Software pipelining is more robust and scalable for loops and programs with complex control flow. It offers high performance with predictable code size and is less sensitive to control flow variations. Trace scheduling can produce good results in well-predicted paths but risks code bloat and is harder to manage for general-purpose programs.

---
### **Final Remarks**

This paper helped me understand that software pipelining can be a practical and effective scheduling technique, especially for loops with complex control flow. Its ability to maintain performance without relying on specialized hardware was notable. While trace scheduling has its own strengths, software pipelining—through hierarchical reduction and modulo variable expansion—offers a more consistent approach for VLIW architectures.

It also reminded me that understanding the architectural features of the target processor is essential when evaluating or designing compiler techniques.

#### Notes
> Inter-iteration data dependency, or recurrences, do not necessarily mean that the code is serialized. This is one 
important advantage that VLIW architectures have over vector machines As long as there are other operations that can 
execute in parallel with the serial computation, a high computation rate can still be obtained (326).

---

### **Reference**
- **[1]** Monica S. Lam. 2003. *Retrospective: Software Pipelining – An Effective Scheduling Technique for VLIW Machines*. SIGPLAN Notices 39, 4 (April 2004), 244–256. ACM. https://doi.org/10.1145/989393.989420
- **[2]** Monica S. Lam. 1988. *Software Pipelining: An Effective Scheduling Technique for VLIW Machines*. In Proceedings of the ACM SIGPLAN '88 Conference on Programming Language Design and Implementation (PLDI '88), 318–328. https://doi.org/10.1145/960116.54022
- **[3]** John L. Hennessy and David A. Patterson. 2017. *Computer Architecture: A Quantitative Approach* (6th ed.). Morgan Kaufmann Publishers Inc., San Francisco, CA, USA.
- **[4]** Roel Jordans and Henk Corporaal. 2015. *High-level Software-Pipelining in LLVM*. In Proceedings of the 18th International Workshop on Software and Compilers for Embedded Systems (SCOPES '15), 97–100. https://doi.org/10.1145/2764967.2771935
- **[5]** Cornel Popescu and Francisc Iacob. 2000. *Model Simulation and Performance Evaluation for a VLIW Architecture*. *Journal of Electrical and Electronics Engineering*, vol. 1, no. 1.
