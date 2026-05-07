---
title: "第八章：AI 推理与训练系统"
weight: 85
description: "面向存储工程师的 AI 推理与训练系统入门：从 Attention、KV Cache、PagedAttention 到 P/D 分离与 Checkpoint，建立理解 AI 场景存储问题的先验知识。"
cascade:
  type: docs
---

近年来大模型推理与训练对存储系统提出了全新需求：KV Cache 跨节点搬运、Checkpoint 高吞吐写入、训练数据的高并发读取……

本章不直接讨论分布式存储论文，而是先帮存储工程师**建立 AI 系统的先验知识**：模型推理时显存里到底放了什么、为什么需要传输、训练时 Checkpoint 又是怎么产生的。理解这些之后，再去读第九章的 AI 场景存储论文会顺畅很多。

{{< section-cards >}}
