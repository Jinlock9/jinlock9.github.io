---
layout: post
title: Compilers by Alex Aiken - 1 Introduction
date: 2025-06-30 20:40 +0800
categories: [Lecture, Compiler]
tags: compiler CS6120
author: jinlock
description: Lecture Note on Stanford Online SOE.YCSCS1
toc: false
published: true
---

# Structure of Compilers
1. Lexical Analysis
2. Parsing
3. Semantic Analysis 
4. Optimization
5. Code Generation

---

### Lexical Analysis
**Lexical analysis** divides program text into "words" or "tokens".

---

### Parsing
* **Parsing** = Diagramming sentences  
* Diagram = tree

ex.
```
if x == y then z = 1; else z = 2;

    x  ==  y   z   1    z   2
    |  |   |   |   |    |   |
    relation   assign   assign
        |        |        |
    predicate   then     else
          \      |      /
           if-then-else
```

---

### Semantic Analysis
* Compilers perform limited **semantic analysis** to catch inconsistencies.
* Programming languages defined strict rules to avoid ambiguities.
* Compilers perform many semantic checks besides variable bindings.

---

### Optimization
Automatically modify programs so that they:
* run faster
* use less memory
* reduce power
* reduce network/database access

Ex.
```
X = Y * 0 is the same as X = 0 (not alway true)
# valid for integer 
# invalid for FP (NaN * 0 = NaN)
```

---

### Code Generation
* Produces assembly code (usually)
* A translation into another language

---

The overall structure of almost every compiler adheres to the outline.
```
# Before
[  L  ][  P  ][S][  O  ][ CG ]
# Thesedays
[L][P][  S  ][    O      ][CG] 
```

---

# The Economy of Programming Languages

### Why are there so many programming languages?
*Application domains have distinctive/conflicting needs.*
* Scientific computing
  - good FP
  - good array
  - parallelism
  - ex. `FORTRAN`
* Business applications
  - persistence
  - report generation
  - data analysis
  - ex. `SQL`
* System programming
  - control of resources
  - real-time constraints
  - ex. `c/c++`

#### Claim: *Programming training is the dominant cost for a programming language*.
* Widely-used languages are slow to change
* Easy to start a new language.
  - productivity > training cost
* Languages adopted to fill a void.
* New languages tend to look like old languages

### What is a good programming language?
There is no universally accepted metric for language design.

* A good language is one people use?