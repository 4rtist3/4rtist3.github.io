---
title: Zero Knowledge and Groth16 Protocol
tags: cryptography blockchain
cover: /assets/images/logo/bg.png
---

# Zero Knowledge Proof
## Introduction

ZKPs are a way to prove a statement is true without revealing anything beyond the truthiness of the statement. It means like "I could prove you that know the fact without tell you a fact".

- I know value X such that `SHA256(X) = 0x7ace...` but don't tell you X.
- I know how to fill a map with three colors so that every two adjacent regions have different colors.
- ...

# Groth16
Consider an arithmetic circuit with addition and multiplication gates over $F$. 

Groth16 is a technique allows you to prove with ZK that you know the values make the arithmetic circuit consistent.

## Quadratic Arithmetic Programs
Assume the given system:

$$\sum_{i=0}^m A_{i,q}w_i . \sum_{i=0}^m B_{i,q}w_i = \sum_{i=0}^m C_{i,q}w_i$$

Where $A_{i,q}$, $B_{i,q}$ and $C_{i,q}$ are public coefficients. If $F$ is big, we use Lagrange Interpolation (see Notes) to compute $A_i(r_q) = A_{i,q}$, similar to $B_i(r_q)$ and $C_i(r_q)$.

A polynomial that satisfies $A_i, B_i, C_i$ could be written as:

$$\sum_{i=0}^m A_i(x)w_i . \sum_{i=0}^m B_i(x)w_i = \sum_{i=0}^m C_i(x)w_i + h(x)z(x)$$

Where  $z(x)$ is **`vanishing polynomial`** or $$ z(x) = \prod_{i = 1}^n (x - r_i)$$ 
# Notes
## Lagrange interpolating
The Lagrange interpolating polynomial is the polynomial $P(x)$ of degree $\leq (n-1)$ that passes through $n$ points. This could be written as:

$$P(x) = \frac{(x - x_2)(x-x_3)...(x-x_n)}{(x_1-x_2)(x_1-x_3)...(x_1 - x_n)}y_1 + \frac{(x - x_1)(x-x_3)...(x-x_n)}{(x_2-x_1)(x_2-x_3)...(x_2 - x_n)}y_2 + ...  + \frac{(x - x_1)(x-x_2)...(x-x_{n-1})}{(x_n-x_1)(x_n-x_2)...(x_n - x_{n-1})}y_n $$


