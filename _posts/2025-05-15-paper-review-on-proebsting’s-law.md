---
layout: post
title: Review - On Proebsting’s Law
date: 2025-05-15 23:00 +0800
categories: [Paper, Review]
tags: compiler CS6120
author: jinlock
description: Reviewing "On Proebsting’s Law"
toc: false
published: true
---

![On Proebsting’s Law](../assets/img/posts/2025-05-15-paper-review-on-proebsting’s-law.png)

To study compilers in a more structured manner, I found a lecture from Cornell University: **CS6120: Advanced Compilers**, a PhD-level computer science course taught by *Adrian Sampson*. Starting today, I plan to review and reflect on what I learn from this course. In the orientation lecture, the professor recommended reading **On Proebsting’s Law** by *Kevin Scott*.

The paper conducts an experiment to examine the validity of Proebsting’s Law.

> **Proebsting’s Law** asserts that improvements to compiler technology double the performance of typical programs every *18 years*.

Proebsting’s observation on the long-term performance impact of compiler optimizations was quite thought-provoking. Compared to Moore’s Law, which states that the capacity of semiconductor ICs doubles every 18 to 24 months, the improvement from compiler technology is considerably smaller. As the author concludes, Proebsting’s Law implies that researchers may be better off focusing on improving programmer productivity rather than relying solely on compiler optimizations.

Of course, Proebsting’s Law was proposed in 1998 and the paper was written in 2001, when Moore’s Law was still considered valid. However, Moore’s Law is no longer applicable today, and the importance of domain-specific architectures (DSAs) and energy efficiency is increasingly recognized. Therefore, I believe the relative importance of even small improvements achieved through compiler optimization has increased.

This paper reminded me not to rely solely on one area of thinking. Although certain approaches in research may yield faster and more visible results, their limitations are often unknown, and we cannot predict what will ultimately become central to system development.



### **Reference**
**[1]** Scott, Kevin. “On Proebsting''s Law.” (2001).