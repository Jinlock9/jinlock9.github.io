---
layout: post
title: Review - Why Systolic Architectures?
date: 2025-02-23 23:20 +0800
categories: [Paper, Computer_Architecture]
tags: computer_architecture systolic_array specical_purpose_architecture
author: jinlock
description: Reviewing "Why Systolic Architectures?"
toc: false
published: true
---

To better understand the architecture of the **Tensor Processing Unit (TPU)**, I decided to start with *Systolic Architecture*.  
The paper I will review is **"Why Systolic Architectures?"** by *H. T. Kung*.

![Why Systolic Architectures?](../assets/img/posts/2025-02-23-paper-review-why-systolic-architecture.png)

The paper first highlights that high-performance, special-purpose computer systems are typically built on an **ad hoc** basis, resulting in a lack of systematic methodologies.   
To address these limitations, it proposes **systolic architecture** as a general guideline for designing such systems.

A **systolic architecture** has several key features:  
- Data flows from memory, passing through multiple processing units before returning to memory.  
- It can be two-dimensional.  
- Data flows at multiple speeds and in multiple directions, unlike classical pipelined systems.  
- **Regularity** makes it easy to implement.  
- **Modularity** allows for easy reconfiguration.  

To understand why systolic architecture is a general solution, we must first examine the key issues in designing special-purpose systems.

### Key Architectural Issues in Designing Special-Purpose Systems 
1. Simple and Regular Design  
   - Special-purpose systems must be cost-effective, with low design costs.  
   - Using simple, repetitive structures reduces complexity and improves scalability.  

2. Concurrency and Communication
   - Faster computation is achieved through concurrency rather than faster individual components.  
   - Algorithms should maximize parallelism using pipelining and multiprocessing.  

3. Balancing Computation with I/O 
   - System performance depends on balancing computation speed with I/O bandwidth.  
   - Overly fast processing without sufficient I/O capacity creates bottlenecks.  
   - Modular designs allow flexibility in adapting to different I/O bandwidth constraints.  

(To be continue...)

### Insights
#### *"I/O and computation imbalance is notable ... fact that I/O interfaces cannot keep up with device speed is discovered only after constructing a high-speed, special purpose device."*
This sentence reminds me of how critical the **I/O-bound bottleneck** can be.   
When I implemented an **MLP Neural Network Accelerator** for a college project, I encountered a similar issue—despite optimizing the computation, the system’s performance was ultimately limited by data transfer through the **I/O interfaces**.   
To work effectively on various **SoCs**, I need to explore different methodologies to mitigate this bottleneck.

### Reference
H. T. Kung, *"Why Systolic Architectures?"* Computer, vol. 15, no. 1, pp. 37–46, Jan. 1982.