---
layout: post
title: Inside the TPU v1 - Architectural Design Principles
date: 2026-10-09 01:40 +0800
categories: [Review, Computer_Architecture]
tags: computer_architecture tensor_processing_unit tpu systolic_array dae_architecture
author: jinlock
description: Exploring Google's Tensor Processing Unit
toc: false
published: true
---

This review explores Google's first custom ASIC for machine learning, the **Tensor Processing Unit (TPU)**, based on the textbook *Computer Architecture: A Quantitative Approach* and Google's TPU v1 architecture paper.

The TPU is a domain-specific accelerator designed by Google to accelerate machine learning workloads, particularly deep learning inference.

This review focuses on the **architectural principles and design decisions** behind the TPU, introducing hardware specifications where they help explain those decisions. All discussions are based on **TPU v1**.

---

### **TPU Architecture Overview**

The TPU v1 was designed as a **coprocessor** connected to the host server through a PCIe I/O bus.

The host sends instructions to the TPU, rather than having the TPU independently fetch instructions from memory.

The architecture consists of dedicated components for matrix computation, data storage, and data movement.

```text
+----------------------+       +----------------------+
|     Host Server      |<----->|    PCIe Gen3 x16     |
+----------------------+       +----------+-----------+
                                          |
                                          v
                               +----------------------+
                               |    Host Interface    |
                               +----------+-----------+
                                          |
                           +--------------+--------------+
                           |                             |
                           v                             v
                +---------------------+       +------------------+
                | Instruction Buffer  |       | Host / UB DMA    |
                |   & Control Logic   |       |   Transfers      |
                +----------+----------+       +--------+---------+
                           :                           |
                     Control Signals                   |
                           :                           v
+----------------------+       +-------------------------------+
|    Weight Memory     |       | Unified Buffer (Activations)  |
|      DDR3 DRAM       |       |           24 MiB              |
+----------+-----------+       +---------------+---------------+
           |                                   |
           v                                   v
+----------------------+       +-------------------------------+
|     Weight FIFO      |------>|      Matrix Multiply Unit     |
+----------------------+       |         256 x 256 PEs         |
                               +---------------+---------------+
                                               |
                                               v
                               +-------------------------------+
                               |         Accumulators          |
                               +---------------+---------------+
                                               |
                                               v
                               +-------------------------------+
                               |      Activation / Pooling     |
                               +---------------+---------------+
                                               |
                                               v
                               +-------------------------------+
                               |         Unified Buffer        |
                               |       (Same 24 MiB UB)        |
                               +-------------------------------+
```
{: .nolineno }

This diagram illustrates the simplified data path of TPU v1. The control logic issues commands to the functional units, while weights and activations follow separate data paths.

The host can also transfer data to and from the Unified Buffer through PCIe using DMA. The Unified Buffer shown at the bottom is the same physical buffer used to supply activations to the Matrix Multiply Unit.

#### **Components (TPU v1)**

- **Matrix Multiply Unit (MMU)**: The computational core of the TPU, containing a 256 × 256 systolic array of 8-bit multiply-accumulate units.
- **Accumulators**: 4 MiB of 32-bit storage used to accumulate partial sums produced by the matrix unit.
- **Activation Hardware**: Supports nonlinear activation functions such as sigmoid and tanh.
- **Weight FIFO**: An on-chip buffer that stages weights before they enter the matrix unit.
- **Weight Memory**: 8 GiB of off-chip DDR3 DRAM used to store model weights.
- **Unified Buffer (UB)**: 24 MiB of on-chip SRAM used to store activations and intermediate results.

One important characteristic of this architecture is the separation of **weight storage** and **activation storage**. Weights are supplied through the Weight FIFO, while activations are provided by the Unified Buffer.

This allows the TPU to organize data movement around the requirements of its matrix computation unit.

---

### **TPU Microarchitecture**

#### **Hiding Latency**

The microarchitecture philosophy of the TPU is to keep the **Matrix Multiply Unit busy**.

Since the MMU performs most of the arithmetic operations, keeping it busy is essential for achieving high throughput.

However, matrix multiplication requires a continuous supply of operands. If the MMU must wait for weights or activations to arrive, its computational resources remain idle.

The TPU addresses this problem by providing separate hardware for different types of operations, allowing some instructions to execute concurrently with matrix multiplication. [1]

For example, while the MMU performs a matrix multiplication, the TPU can prepare weights for subsequent operations.

The key idea is to **overlap data movement with computation**, rather than executing every operation sequentially.

One concrete example is **double-buffered weight storage**.

The Matrix Multiply Unit supports two weight tiles, allowing one tile to be used for computation while the next tile is loaded.

Loading a complete 256 × 256 weight tile takes approximately 256 cycles. By overlapping this loading process with computation, the TPU can hide much of the weight-loading latency when the computation lasts long enough. [2]

This reduces the time during which the MMU is idle and improves its overall utilization.

#### **Even More ILP: Decoupled Access/Execute**

The `Read_Weights` instruction follows the **decoupled access/execute** philosophy: it can complete after issuing the addresses for a weight transfer, before the weights have actually arrived from Weight Memory. [1]

The **Decoupled Access/Execute (DAE)** architecture separates memory access and computation into two independent instruction streams.

In a conventional DAE architecture, an access processor handles memory operations, while an execute processor performs computations. The two processors communicate asynchronously through FIFO queues. [3]

```text
+------------------------------------------------------------------+
|                        INSTRUCTION STREAM                        |
+--------------------------------+---------------------------------+
                                 |
              +------------------+------------------+
              |                                     |
              v                                     v
+----------------------------+    +----------------------------+
|      ACCESS PROCESSOR      |    |      EXECUTE PROCESSOR     |
|       (Address Gen)        |    |        (ALU / MAC)         |
+-------------+--------------+    +--------------+-------------+
              |                                  ^
              |                                  |
              |        COMMUNICATION FIFOs       |
              |     +-----------------------+    |
              +---->|    Data from Memory   |----+
              |     +-----------------------+    |
              +---->| Control / Predicates  |----+
              |     +-----------------------+    |
              +<----|      Store Data       |<---+
              |     +-----------------------+
              v
+----------------------------+
|        MAIN MEMORY         |
|           DRAM             |
+----------------------------+
```
{: .nolineno }

The diagram is a simplified conceptual representation of DAE, not the actual TPU v1 microarchitecture. Store data passes from the execute side through a FIFO to the access side. The access side then writes it to memory.

The TPU v1 does not implement this conventional DAE architecture with two independent instruction streams. However, its `Read_Weights` instruction adopts a similar principle.

The instruction can complete after issuing the memory addresses, without waiting for the actual weights to arrive from DRAM.

This decouples instruction execution from the completion of the corresponding memory access.

In the TPU, the Weight FIFO plays an important role in this process:

- **Memory Access**: The weight memory interface fetches weights from off-chip DRAM.
- **Buffering**: The Weight FIFO temporarily stores fetched weights.
- **Execution**: The Matrix Multiply Unit receives weights from the FIFO for loading into its processing elements.

By fetching weights ahead of their consumption, the TPU can overlap memory access with matrix computation.

However, the FIFO does not eliminate memory latency. It only helps hide that latency when sufficient data has been prefetched. If the required weights are not available in time, the MMU may have to wait.

Thus, the main benefit of this DAE-inspired design is **reducing unnecessary synchronization between memory access and computation**, allowing the MMU to maintain higher utilization.

---

#### **Systolic Architecture**

Even if memory latency is successfully hidden, another important problem remains: **the cost of moving data**.

Matrix multiplication requires a large number of multiply-accumulate operations. Although each arithmetic operation is relatively inexpensive, repeatedly reading operands from memory consumes significant energy and bandwidth.

To address this problem, the TPU uses a **systolic array**, a two-dimensional grid of processing elements (PEs) that perform computations and pass data between neighboring elements.

The TPU v1 contains a 256 × 256 systolic array with 65,536 8-bit multiply-accumulate units, capable of performing up to **131,072 arithmetic operations per cycle** when multiplication and addition are counted separately.

At its nominal 700 MHz clock frequency, this corresponds to approximately **92 TOPS** of peak 8-bit arithmetic throughput. [2]

##### **Weight Loading and Systolic Execution**

The TPU v1 uses a weight-stationary form of systolic execution.

The computation can be understood in two phases:

1. **Weight Loading**: Weights are shifted vertically into the array. They are loaded into a second weight register in each PE while the current tile is still computing.
2. **Matrix Computation**: The loaded weights are reused while activations move horizontally and partial sums propagate vertically through the array.

During computation, each PE performs a multiply-accumulate operation using its loaded weight, the incoming activation, and the incoming partial sum.

The following diagram provides a simplified conceptual view of the systolic dataflow.

```text
                  Weight Load (Preload Phase)
                       |       |       |
                       v       v       v
                     +----+  +----+  +----+
Activation (Row 0) ->| PE |->| PE |->| PE |--->
                     +----+  +----+  +----+
                       |       |       |
                       v       v       v
                     +----+  +----+  +----+
Activation (Row 1) ->| PE |->| PE |->| PE |--->
                     +----+  +----+  +----+
                       |       |       |
                       v       v       v
                     +----+  +----+  +----+
Activation (Row 2) ->| PE |->| PE |->| PE |--->
                     +----+  +----+  +----+
                       |       |       |
                       v       v       v
                          Partial Sums
                           (Matrix C)

       Each PE holds a weight during the compute phase.
       Vertical arrows represent partial-sum propagation
       during computation, not weight movement.
```
{: .nolineno }

During the computation phase, the weights remain available locally in the PEs and can be reused across multiple activation values.

The activations are also staggered in time as they enter the array, causing computation to progress as a **diagonal wavefront**.

This is an important distinction: the computation wavefront moves diagonally, but partial sums propagate vertically between processing elements.

##### **Spatial Data Reuse**

The important characteristic of this architecture is **spatial data reuse**.

In conventional matrix multiplication, the same operands may be fetched repeatedly from memory for different arithmetic operations.

In the TPU's systolic array, weights are reused locally while activations and partial sums are transferred between neighboring PEs.

This reduces repeated accesses to the Unified Buffer and increases the amount of computation performed per memory access.

As H. T. Kung explained, systolic architectures can increase computational throughput without requiring a proportional increase in memory bandwidth. [4]

Increasing the number of arithmetic units alone does not guarantee higher performance. Without sufficient memory bandwidth, additional computation units may remain idle.

Systolic execution addresses this problem by reducing the bandwidth required to sustain a large number of arithmetic operations.

Consequently, it improves both **computational throughput and energy efficiency**.

##### **Implications for Software**

Although software does not directly control individual PEs in the systolic array, it must still account for the execution characteristics of the Matrix Multiply Unit.

For example, loading a complete weight tile requires approximately 256 cycles, and the array has pipeline fill and drain latency associated with its systolic execution.

These costs influence how matrix operations should be organized and scheduled.

From a compiler perspective, this means that efficient execution requires more than simply generating matrix multiplication instructions.

The compiler and runtime must also consider **data movement, operation scheduling, and hardware utilization** to minimize idle cycles and exploit the available computational throughput.

---

### **Summary**

The TPU v1 demonstrates how a domain-specific architecture can achieve high computational throughput and energy efficiency by addressing the bottlenecks of its target workloads.

Three architectural principles are particularly important:

- **Latency Hiding and Instruction-Level Parallelism**: The TPU overlaps matrix multiplication with other operations, reducing idle cycles in its computational core. Double-buffered weight storage allows weight loading to overlap with ongoing computation.

- **Decoupled Memory Access**: The asynchronous behavior of `Read_Weights`, together with the Weight FIFO, allows memory access to overlap with computation and reduces unnecessary synchronization.

- **Systolic Execution and Data Reuse**: The 256 × 256 systolic array reuses loaded weights and passes activations and partial sums between neighboring processing elements, reducing repeated memory accesses and the bandwidth required for matrix multiplication.

These principles address different aspects of the same problem: **keeping the Matrix Multiply Unit supplied with data while minimizing the cost of moving that data**.

The TPU v1 illustrates that high performance does not come solely from increasing computational capacity. It also requires an architecture that can efficiently organize instruction execution, memory access, and data movement around the computational workload.

---

### **References**

- **[1]** John L. Hennessy and David A. Patterson. 2017. *Computer Architecture: A Quantitative Approach* (6th ed.). Morgan Kaufmann Publishers Inc., San Francisco, CA, USA.
- **[2]** Norman P. Jouppi et al. 2017. *In-Datacenter Performance Analysis of a Tensor Processing Unit*. Proceedings of the 44th Annual International Symposium on Computer Architecture (ISCA '17), pp. 1–12. https://doi.org/10.1145/3079856.3080246
- **[3]** James E. Smith. *Decoupled access/execute computer architectures*. SIGARCH Computer Architecture News 10, 3 (April 1982), 112–119. https://doi.org/10.1145/1067649.801719
- **[4]** H. T. Kung. *Why Systolic Architectures?* Computer, vol. 15, no. 1, pp. 37–46, Jan. 1982.