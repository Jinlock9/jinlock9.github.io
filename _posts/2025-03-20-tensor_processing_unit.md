---
layout: post
title: Tensor Processing Unit (TPU)
date: 2025-03-21 22:00 +0800
categories: [Review, Computer_Architecture]
tags: computer_architecture tensor_processing_unit tpu
author: jinlock
description: Exploring Google's Tensor Processing Unit
toc: false
published: true
---

The topic I am going to explore this time is Google's first custom ASIC DSA, **Tensor Processing Unit (TPU)**. The book I reviewed provides information only about **TPU v1**, but since my review focuses more on the **ideas and fundamental architecture** rather than the specific hardware specifications, please note that my review is based on **TPU v1**.

Tensor Processing Unit (TPU) is a custom-designed application-specific integrated circuit (ASIC) developed by Google specifically for accelerating machine learning (ML) workloads, particularly deep learning inference and training.

---

### **TPU Architecture**

The TPU was designed as a **coprocessor** on the PCIe I/O bus. The host server sends instructions over the PCIe bus directly to the TPU, so the TPU does not fetch instructions itself.

#### **Components (TPU v1)**
- **Matrix Multiply Unit**: Heart of the TPU, containing a 256 × 256 array of 8-bit ALUs.
- **Accumulator**: 4 MiB of 32-bit registers. Collects the 16-bit products from the Matrix Multiply Unit.
- **Activation Hardware**: Computes nonlinear functions such as sigmoid and tanh.
- **Weight FIFO**: On-chip buffer that stages weights before they are used by the matrix unit.
- **Weight Memory**: Off-chip 8 GiB DRAM from which the Weight FIFO reads.
- **Unified Buffer**: 24 MiB on-chip memory that stores input activations, intermediate results, and output data.

---

### **TPU Instruction Set Architecture (TPU v1)**

TPU instructions follow the **CISC** tradition. The TPU does not have a program counter or branch instructions, as instructions are **sent directly from the host CPU** rather than fetched or scheduled internally.

#### **Key Instructions**
1. `Read_Host_Memory`
2. `Read_Weights`
3. `MatrixMultiply` / `Convolve`
4. `Activate`
5. `Write_Host_Memory`

---

### **TPU Microarchitecture**

The TPU microarchitecture is optimized to **keep the Matrix Multiply Unit (MXU) busy** at all times. It achieves this by overlapping matrix operations with other instructions and using **dedicated execution hardware** for different instruction types. This parallelism is further enhanced through the **decoupled access/execute** model—for example, `Read_Weights` can proceed after issuing an address without waiting for the data to arrive.

At the core of the TPU is a **systolic array**, a 2D grid of multiply-accumulate (MAC) units. In this structure, **data flows like a wave**:
- Inputs come from the **left**.
- Weights are loaded from the **top**.
- Partial results move **diagonally** through the array to form the final output.

This **systolic execution** minimizes expensive memory reads and writes by passing data between neighboring MAC units. Inputs are read **once**, and outputs are written **once**, greatly improving energy efficiency and throughput.

Although software is **unaware of the systolic structure**, for optimal performance it must account for the **latency** of the matrix unit when scheduling instructions.

---

### **TPU Programming Model (TPU v2 and later)**
TPUs are tightly integrated with TensorFlow, but also support JAX and PyTorch via XLA (Accelerated Linear Algebra). The key steps to use TPUs in TensorFlow include:

- Defining a TensorFlow computation graph.
- Converting the model to XLA format for execution on TPU hardware.
- Running computations using TPU hardware acceleration.

---

### **Review**
Exploring the TPU was fascinating because the way Google identified the problem, designed, verified, and deployed the TPU while balancing cost and time constraints reminded me of what I vaguely learned in the System-on-Chip Design course. It also made me realize the importance of SoC design. I agreed with Google’s decision to design the TPU as a coprocessor rather than an independent processor. This approach made sense, especially for a company like Google that is not a chip design company, as it allowed them to leverage existing systems more effectively. Because of this, I now think I should explore various IPs beyond conventional computer architecture—such as the PCIe Bus, AMBA, and others.

Also, I already knew that the systolic architecture is adopted in the TPU, so I reviewed it in advance. However, I was pleasantly surprised to see that the DAE (Decoupled Access/Execute) architecture philosophy is reflected in the TPU's microarchitecture. It was impressive that the TPU incorporates philosophies and advantages from various architectural sources. To be honest, I didn't consider DAE architecture very important before, but now I think I should stay open-minded to different ideas.

I like the TPU's **microarchitecture philosophy**, which states that instead of executing instructions sequentially like a CPU, the TPU hides the execution of memory, activation, and weight operations behind ongoing matrix multiplications. This ensures maximum utilization of the MXU, the most critical hardware component for deep learning workloads. Ultimately, I believe this embodies the **fundamental principle of parallel computing**.

One of the main reasons I wanted to study TPU architecture was to understand its memory interactions. I realized that by adopting **systolic architecture** and **on-chip memory**, the TPU achieves a highly efficient structure that significantly improves memory bandwidth and reduces latency. Additionally, since it functions as a **coprocessor**, I learned that it does not contribute to the memory bottleneck of the host CPU—a realization that corrected my previous misconception that matrix operations could be a major source of register pressure. This was truly a moment of breaking free from my ignorance—haha!

### **Reference**
- **[1]** John L. Hennessy and David A. Patterson. 2017. Computer Architecture, Sixth Edition: A Quantitative Approach (6th. ed.). Morgan Kaufmann Publishers Inc., San Francisco, CA, USA.