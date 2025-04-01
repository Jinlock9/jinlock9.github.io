---
layout: post
title: Software Pipelining
date: 2025-04-01 23:00 +0800
categories: [Review, Compiler]
tags: compiler instruction_level_parallelism software_pipelining
author: jinlock
description: Studying Software Pipelining
toc: false
published: true
---

### Background

As I explore VLIW processors, it is all about finding sufficient (and ideally high) Instruction Level Parallelism (ILP) to fully utilize the benefits of VLIW. The papers and textbooks I have read to learn about VLIW processors usually introduce *trace scheduling* as the primary solution for uncovering ILP (which I plan to review soon). 

However, this time, I read a short paper titled *Software Pipelining: An Effective Scheduling Technique for VLIW Machines*. The author argues that "software pipelining is effective on VLIW without complicated hardware support" [1]. Since I believe it is important to understand various ways to effectively utilize VLIW processors, I decided to dig into **software pipelining**.

Although I need to explore it more deeply, I’ve already noticed that compilation using software pipelining can be quite challenging due to the complexity of balancing overhead and efficiency, as well as managing registers effectively [2]. To address these challenges, some architectures have added extra hardware support [2]. However, I believe that increasing hardware complexity goes against the core philosophy of VLIW architecture, which is to eliminate sophisticated instruction scheduling logic in hardware [4]. This makes me even more curious about how software pipelining can be effective on VLIW machines without relying on complex hardware support.

But for now, let’s study software pipelining from the basics~!

---

### Software Pipelining

---

### **Reference**
- **[1]** M. Lam. 1988. Software pipelining: an effective scheduling technique for VLIW machines. SIGPLAN Not. 23, 7 (July 1988), 318–328. https://doi.org/10.1145/960116.54022
- **[2]** John L. Hennessy and David A. Patterson. 2017. Computer Architecture, Sixth Edition: A Quantitative Approach (6th. ed.). Morgan Kaufmann Publishers Inc., San Francisco, CA, USA.
- **[3]** Roel Jordans and Henk Corporaal. 2015. High-level software-pipelining in LLVM. In Proceedings of the 18th International Workshop on Software and Compilers for Embedded Systems (SCOPES '15). Association for Computing Machinery, New York, NY, USA, 97–100. https://doi.org/10.1145/2764967.2771935
- **[4]** Popescu, Cornel and Francisc Iacob. “MODEL SIMULATION AND PERFORMANCE EVALUATION FOR A VLIW ARCHITECTURE.” (2000).