---
title: Zero Knowledge and Groth16 Protocol
tags: cryptography blockchain
cover: /assets/images/logo/bg.png
---

# Zero Knowledge Proof

ZKPs are a way to prove a statement is true without revealing anything beyond the truthiness of the statement. It means like "I could prove you that know the fact without tell you a fact".

- I know value X such that `SHA256(X) = 0x7ace...` but don't tell you X.
- I know how to fill a map with three colors so that every two adjacent regions have different colors.
- ...

The ZKPs have three core properties:
- **Completeness:** Given the statement and the witness, the prover can convince the verifier.
- **Soundness:** A malicious/fake prover cannot convince the verifier of false statement.
- **Zero-Knowledge:** The proof does not reveal anything but the truth of the statement, in particular it does not reveal the prover's witness.

# Groth16
Consider an arithmetic circuit with addition and multiplication gates over $\mathbb{F}$. 

Groth16 is a technique allows you to prove with ZK that you know the values make the arithmetic circuit consistent.

## Quadratic Arithmetic Programs
Assume the given system:

$$\sum_{i=0}^m A_{i,q}w_i . \sum_{i=0}^m B_{i,q}w_i = \sum_{i=0}^m C_{i,q}w_i$$

Where $A_{i,q}$, $B_{i,q}$ and $C_{i,q}$ are public coefficients. If $\mathbb{F}$ is big, we use Lagrange Interpolation (see Notes) to compute $A_i(r_q) = A_{i,q}$, similar to $B_i(r_q)$ and $C_i(r_q)$.

A polynomial that satisfies $A_i, B_i, C_i$ could be written as:

$$\sum_{i=0}^m A_i(x)w_i . \sum_{i=0}^m B_i(x)w_i = \sum_{i=0}^m C_i(x)w_i + h(x)z(x)$$

for some degree $n − 2$ quotient polynomial $h(x)$ where $n$ is the degree of $z(x)$, with $z(x)$ is **vanishing polynomial** or we could write $z(x)$ as: 

$$z(x) = \prod_{i = 1}^n (x - r_i)$$ 

## Non-Interactive Zero-Knowledge (NIZK) argument

Let $R$ be efficiently decidable relation. If $(\phi,\omega) \in R$ then we can denote $\phi$ as the statement and $\omega$ as the witness. A NIZK has:

- $(\sigma,\tau) \longleftarrow \text{Setup}(R)$, produces common reference string (CRS) $\sigma$ and a simulation trapdoor $\tau$ for relation $R$.
- $\pi \longleftarrow \text{Prove}(R,\sigma,\phi,\omega)$ providing proof of knowledge.
- $0/1 \longleftarrow \text{Verify}(R,\sigma,\phi,\pi)$ which verifies the proof.
- $\pi \longleftarrow \text{Sim}(R,\tau,\phi)$ The simulator takes input as simulation trapdoor and the statement.

## Non-Interactive Linear Proof (NILP)

NILPs are defined relative to a relation generator $R$, where we assume the relations specify a finite field $\mathbb{F}$, and work as follows:

- $\sigma \longleftarrow \text{Setup}(R)$ work as Setup in NIZK.
- $\pi \longleftarrow \text{Prove}(R,\sigma,\phi,\omega)$ The prover operates in two state:
    - First, It runs $\Pi \longleftarrow \text{ProofMatrix}(R,\phi,\omega)$ where $\text{ProofMatrix}$ is a probabilistic polynomial time algorithm that generates a matrix $\Pi \in \mathbb{F}^{k*m}$
    - Then it compute the proof as $\pi =\Pi\sigma$.

- $0/1 \longleftarrow \text{Verify}(R,\sigma,\phi,\pi)$ which verifies the proof via evaluation of multivariate polynomials, and checking whether the result is 0.

Assume $w(0) = 1$ and $w(1), w(2),... w(l)$ are public and we want to prove $w(l + 1), w(l + 2),... w(m)$. To construct NILP for quadratic arithmetic program of Groth16, we first start at Setup state where we select $\alpha,\beta,\delta,x \in \mathbb{F}^{*}$ randomly. Set $\tau = (\alpha,\beta,\delta,x)$ and the CRS $\sigma$ contains:

- $\beta,\delta$
- $x^i \text{ for } 0 \leq i \leq 2n-2$
- $\alpha x^i \text{ for } 0 \leq i \leq n$
- $\beta x^i \text{ for } 0 \leq i \leq n$
- $\frac{\beta A_i(x) + \alpha B_i(x) + C_i(x)}{\delta} \text{ for } l + 1 \leq i \leq m$
- $\frac{x^i z(x)}{\delta} \text{ for } 0 \leq i \leq n-2$


# Notes
### Lagrange interpolating
The Lagrange interpolating polynomial is the polynomial $P(x)$ of degree $\leq (n-1)$ that passes through $n$ points. This could be written as:

$$P(x) = \frac{(x - x_2)(x-x_3)...(x-x_n)}{(x_1-x_2)(x_1-x_3)...(x_1 - x_n)}y_1 + \frac{(x - x_1)(x-x_3)...(x-x_n)}{(x_2-x_1)(x_2-x_3)...(x_2 - x_n)}y_2 + ...  + \frac{(x - x_1)(x-x_2)...(x-x_{n-1})}{(x_n-x_1)(x_n-x_2)...(x_n - x_{n-1})}y_n $$
