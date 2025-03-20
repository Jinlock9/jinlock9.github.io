---
layout: post
title: Review - Decoupled Access/Execute Computer Architectures
date: 2025-03-09 22:50 +0800
categories: [Paper, Computer_Architecture]
tags: computer_architecture dae_architecture
author: jinlock
description: Reviewing "DECOUPLED ACCESS/EXECUTE COMPUTER ARCHITECTURES"
toc: false
published: true
---

For my capstone project, I transformed a single-scalar RISC-V pipelined processor into an out-of-order pipelined processor using Tomasulo's Algorithm, which requires a reservation station and a robust reorder buffer. To enhance out-of-order execution, our team implemented various features, including multiple functional units, speculative execution with branch misprediction recovery, and instruction prefetching. While we achieved significant performance improvements, we also observed that this approach demands substantial hardware resources.  

As I explored different computer architectures, I came across *Decoupled Access/Execute Architectures*, which appear to reduce the hardware burden compared to Tomasulo’s Algorithm. This intrigued me, leading me to further investigate the topic. Therefore, the paper I am reviewing this time is **DECOUPLED ACCESS/EXECUTE COMPUTER ARCHITECTURES** by *James E. Smith*.

![DECOUPLED ACCESS/EXECUTE COMPUTER ARCHITECTURES](../assets/img/posts/2025-03-09-paper-review-decoupled-access-execute-arch.jpg)

### **Review**
The core idea of the Decoupled Access/Execute (DAE) architecture is to separate memory access and execution into two independent instruction streams — one handling memory operations and the other performing computations. These two streams communicate asynchronously via FIFO queues, allowing execution to proceed without being stalled by memory access delays. 

At first glance, I thought it resembled a general architecture with a separate addressing unit, but it turned out to be much more sophisticated and advanced. Although the paper states that DAE reduces the programmer’s burden compared to other array processors, having recently worked on compiler development, I couldn’t help but feel that this reduction wasn’t quite enough (joke). 

---

### **Summary**
#### **1. Introduction**
Traditional scalar computer architectures suffer from instruction issue bottlenecks (*"instruction pass at the maximum rate of one per clock period"*) and memory communication delays. The *Decoupled Access/Execute (DAE)* architecture aims to mitigate these issues by separating operand access from execution. DAE consists of two independent instruction streams communitcating via queues, reducing programmer burden and increasing performance.

#### **2. Architecture Overview**
The architecture comprises two processors:

- **Access Processor (A-Processor)**: Handles memory accesses, address calculations, and data transfers.
- **Execute Processor (E-Processor)**: Executes operations using operands provided by the A-processor.

Data flows between the processors using **FIFO queues**:

- **Access to Execute Queue (AEQ)**: Sends operands from A-processor to E-processor.
- **Execute to Access Queue (EAQ)**: Returns results from E-processor to A-processor.

Memory stores are handled asynchronously to improve execution speed.

#### **3. Handling Memory Stores**
Stores are issued as soon as addresses are computed, without waiting for data. This approach allows loads to proceed without being blocked by pending stores. A **Write Address Queue (WAQ)** holds store addresses until corresponding data is available. A potential issue is store-load conflicts, which can be resolved via *interlocks* or *associative comparison* in hardware.

#### **4. Conditional Branch Instructions**
The A-Processor should determine as many conditional branches as possible instead of the E-Processor (Execute Processor). This approach reduces the E-Processor's dependency on the A-Processor, allowing the A-Processor to run ahead without being blocked.  

Branches must be synchronized between A-processor and E-processor.  

Two additional queues:

- **E to A Branch Queue (EABQ)**: Sends branch outcomes from E-processor to A-processor.
- **A to E Branch Queue (AEBQ)**: Sends branch decisions from A-processor to E-processor.

This ensures efficient handling of conditional branches, allowing the A-processor to run ahead when possible.

#### **5. Queue Architecture and Implementation**
The queues are implemented as circular buffers within the processors: *"make some of the general purpose registers the queue heads and tails"*. Register-based queues eliminate the need for special instructions. The **queue full/empty status** is tracked using **busy bits**, preventing deadlock.

#### **6. Performance Improvement**
Performance is evaluated using the *Lawrence Livermore Loops* benchmark. The DAE approach acheives an average speedup fo *1.71x*, with some loops reaching *2.5x* improvement. The decoupling allows more efficient instruction issuing compared to traditional scalar architectures.

#### **7. Single Instruction Stream DAE Architecture**
The dual-instruction stream model is effective but introduces some disadvantages:
- The programmer (and/or compiler writer) must deal with two interacting instruction streams.
- Two separate instruction fetch and decode units are needed.

Alternative approach: **Interleaved Instruction Stream**
  - where A and E instructions are mixed into a single stream.
  - Simplifies software development.
  - But, reintroduces the one instruction per clock period bottlenech in the IF/ID pipeline.

#### **8. Deadlock Issues**
Deadlocks occur when **both queues are either full or empty**, causing processors to stall. To prevent, proper **instruction interleaving** ensures that a send operation always precedes a receive operation. Also, a well-structured program guarantees deadlock-free execution.

#### **9. Conclusions**
DAE architectures offer significant performance improvements while reducing programming complexity.

---

### **Insights**
#### *"need for processors that can diminish the effects of increased memory communication time (112)."*
#### *"By architecturally decoupling data access from execution, it is possible to construct implementations that provide much of the performance improvement offered by complex issuing methods, but without significant design complexity. In addition in can allow considerable memory communication delay to be hidden (112)."*
#### *"The issuing stores before data is available is an important factor in improving performance, because it allows load instructions to be issued without waiting for previous store instructions (114)"*
#### *"By using two processors a speedup of greater than two is achieved because the issue logic in a pipelined processor typically spends more time waiting to issue instructions than actually issuing them (117)."*

---

### **Reference**
James E. Smith. 1982. Decoupled access/execute computer architectures. SIGARCH Comput. Archit. News 10, 3 (April 1982), 112–119. https://doi.org/10.1145/1067649.801719