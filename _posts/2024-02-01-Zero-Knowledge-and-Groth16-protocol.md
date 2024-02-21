---
title: Zero Knowledge and Groth16 Protocol
tags: cryptography blockchain
cover: /assets/images/logo/bg.png
---

# Zero Knowledge Proof
## Introduction

ZKPs are a way to prove a statement is true without revealing anything beyond the truthiness of the statement. It means like "I could prove you that know the fact without tell you a fact".

- I know value X such that SHA256(x) = 0x7ace... but don't tell you X.
- I know how to fill a map with three colors so that every two adjacent regions have different colors.
- ...

# Groth16

Groth16 is a technique allows you to prove with ZK that you know the values make the arithmetic circuit consistent.