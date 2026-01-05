---
title: "大型基础模型(LFMs)的检查点保存: ByteCheckPoint"
weight: 1
---

{{< callout type="info" >}}
✋🏻😭✋🏻 本小节编辑中 ✍️✍️✍️
{{< /callout >}}

一个在推理训练框架和 DFS 之间的中间层系统，实现了与并行无关的高性能检查点存取。

看完本文，我们将大概理解大模型推理训练用户：

- 为什么重视 Checkpoint 的存取效率?
- 一个中间层的出现，解决了原来 Checkpoint 的哪些问题?
- 存储 Checkpoints 是在存什么?

另外，我们将延伸思考供所有人参考：

- 刻意练习：如何对 Checkpoint 用户需求做优化?
- 刻意练习：论文中 Checkpoint 存储加速为什么是有效的?

## 背景和必要概念


## 系统设计


## 刻意练习：如何对 Checkpoint 用户需求做优化


## 小结

