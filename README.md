
<div align="center">

# 分布式存储漫游指南

<p align="center">
  <em>分布式存储技术探索之旅 by SPtuan</em>
</p>

<div align="center">

<picture>
  <img src="static/banner.jpg" alt="分布式存储漫游指南" width="330" >
</picture>

</div>

<p align="center">
  <a href="https://storage-memo.steinslab.io/">
    <img src="https://img.shields.io/badge/📖_在线阅读-4A90E2?style=for-the-badge&logoColor=white" alt="在线阅读">
  </a>
  <a href="https://github.com/sptuan/dist-storage-memo">
    <img src="https://img.shields.io/github/stars/sptuan/dist-storage-memo?style=for-the-badge&logo=github&color=yellow" alt="GitHub Stars">
  </a>
</p>

---

</div>

## 📚 关于本书

本漫游指南从现代数据中心硬件、单机 IO 性能出发，逐步探索分布式的存储产品的现状和技术点。

本电子书是笔者从事分布式存储研发经历的小结。带着曾有的疑惑，从新手村重新出发，带有主观色彩。

笔者在学习过程中，受益于诸多开发者的博客文章和技术讨论，故自己也记录下来，希望能和读者多多交流学习。

## 🎯 适合读者

<table>
<tr>
<td width="50%">

**开发者**
- 分布式系统开发者
- 存储系统工程师
- 存储应用开发者

</td>
<td width="50%">

**学习者**
- 对存储技术感兴趣的学生和研究者
- 希望了解分布式存储架构的技术人员
- 正在准备面试材料的求职者

</td>
</tr>
</table>

## 🚀 快速开始

### 📖 在线阅读

<div align="center">

**[👉 立即开始阅读 👈](https://storage-memo.steinslab.io/)**

</div>

### 🛠️ 本地构建

```bash
# 克隆仓库
git clone https://github.com/sptuan/dist-storage-memo.git
cd dist-storage-memo

# 安装 Hugo (以 macOS 为例)
brew install hugo

# 启动本地服务器
hugo server -D

# 构建静态文件
hugo
```

## 📖 目录结构

```
📁 分布式存储漫游指南
├── 🔧 硬件基础
├── 📝 同步 I/O (Sync I/O)
│   ├── 基础概念
│   ├── Direct I/O
│   ├── Memory Mapped I/O
│   └── 线程池模型
├── ⚡ 异步 I/O (Async I/O)
│   ├── Linux AIO
│   ├── io_uring
│   └── Golang 磁盘 I/O
├── 🌐 分布式基础 (Dist-101)
│   ├── CAP 理论
│   ├── 时间与时钟
│   ├── 分区与副本
│   └── 混沌工程
├── 🛠️ 存储引擎系统设计
│   ├── 复制模型和优缺点
│   ├── 分区模型和优缺点
│   └── 数据安全性模型
└── 🗃️ 元数据系统设计
    └── NoSQL 数据库

绝赞更新中……
```

## 🤝 贡献指南

<div align="center">

| 🐛 发现错误 | 💡 提出建议 | 📝 内容贡献 | ⭐ 点赞支持 |
|:---:|:---:|:---:|:---:|
| [提交 Issue](https://github.com/sptuan/dist-storage-memo/issues) | [功能请求](https://github.com/sptuan/dist-storage-memo/issues) | [Pull Request](https://github.com/sptuan/dist-storage-memo/pulls) | [⭐ Star 项目](https://github.com/sptuan/dist-storage-memo) |

</div>

## 👨‍💻 关于作者

<div align="center">

<img src="https://github.com/sptuan.png" width="100" height="100" style="border-radius: 50%;" alt="SPtuan">

**SPtuan**

*一名普通的工程师，最大的愿望是度过平静的时光*

<p align="center">
  <a href="https://github.com/sptuan">
    <img src="https://img.shields.io/badge/GitHub-sptuan-blue?style=flat-square&logo=github" alt="GitHub">
  </a>
  <a href="https://steinslab.io">
    <img src="https://img.shields.io/badge/Blog-steinslab.io-green?style=flat-square&logo=blogger" alt="Blog">
  </a>
  <a href="mailto:sptuan@steinslab.io">
    <img src="https://img.shields.io/badge/Email-sptuan@steinslab.io-red?style=flat-square&logo=gmail" alt="Email">
  </a>
</p>

</div>

---

<div align="center">

### 📞 联系交流

💬 **技术讨论** · 📝 **内容建议** · 🐛 **问题反馈**

<p>
  <strong>Email:</strong> sptuan#steinslab.io <em>(# 替换为 @)</em><br>
  <strong>GitHub:</strong> <a href="https://github.com/sptuan/dist-storage-memo/issues">Issues & Discussions</a>
</p>

#### 👋 分布式存储交流闲聊群

<picture>
  <img src="https://static.zdfmc.net/imgs/2025/10/0d8fc3f543265714.png" alt="WeChat" width="300">
</picture>

世界很大，圈子很小！
让世界更热闹一些吧 (球球了)！
所有的读者都大大大欢迎！
您的反馈是我前进的动力！
WeChat: dangotech

### ⭐ 如果这个项目对您有帮助，请考虑给个 Star！

<a href="https://github.com/sptuan/dist-storage-memo">
  <img src="https://img.shields.io/github/stars/sptuan/dist-storage-memo?style=social" alt="GitHub Stars">
</a>

</div>


