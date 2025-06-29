---
title: 文档
cascade:
  type: docs
---

# 分布式存储漫游指南

5202 年，我们能享受到怎样的存储硬件性能？

本漫游指南从数据中心硬件、单机性能出发，到构建分布式的对象存储产品的见闻和思考。
系列文章是笔者从事对象存储研发经历的小结。从自己曾有的疑惑出发，收集资料和思考形成，希望能和读者多多交流学习。

{{< callout type="info" >}}
📖 本书开源免费，源代码托管在 [GitHub](https://github.com/sptuan/dist-storage-mono)，欢迎 Star 和贡献！

<div class="github-star-button" style="margin: 10px 0;">
  <a href="https://github.com/sptuan/dist-storage-mono" target="_blank" style="display: inline-flex; align-items: center; text-decoration: none; background: #24292e; color: white; padding: 8px 16px; border-radius: 6px; font-size: 14px; font-weight: 500; transition: background-color 0.2s;">
    <svg height="16" width="16" viewBox="0 0 16 16" style="margin-right: 8px;" fill="currentColor">
      <path d="M8 .25a.75.75 0 01.673.418l1.882 3.815 4.21.612a.75.75 0 01.416 1.279l-3.046 2.97.719 4.192a.75.75 0 01-1.088.791L8 12.347l-3.766 1.98a.75.75 0 01-1.088-.79l.72-4.194L.818 6.374a.75.75 0 01.416-1.28l4.21-.611L7.327.668A.75.75 0 018 .25z"/>
    </svg>
    Star
    <span class="github-stars-count" style="margin-left: 8px; background: rgba(255,255,255,0.25); padding: 2px 8px; border-radius: 3px; font-weight: 600;">
      <span style="opacity: 0.7;">...</span>
    </span>
  </a>
</div>

<script>
(async function() {
  try {
    const response = await fetch('https://api.github.com/repos/sptuan/dist-storage-mono');
    const data = await response.json();
    const starsElement = document.querySelector('.github-stars-count');
    if (starsElement && data.stargazers_count !== undefined) {
      starsElement.textContent = data.stargazers_count.toLocaleString();
    }
  } catch (error) {
    console.error('Failed to fetch GitHub stars:', error);
    const starsElement = document.querySelector('.github-stars-count');
    if (starsElement) {
      starsElement.innerHTML = '<span style="opacity: 0.7;">⭐</span>';
    }
  }
})();
</script>

<style>
.github-star-button a:hover {
  background: #2c3e50 !important;
}
</style>
{{< /callout >}}

## 章节目录

{{< cards >}}
  {{< card link="chapter-01" title="第一章：硬件篇" subtitle="2025年了，存储硬件啥样了？" >}}
  {{< card link="chapter-02" title="第二章：单机篇" subtitle="我应怎样在单机上读写数据？" >}}
{{< /cards >}}

### 即将推出

更多章节正在撰写中，敬请期待：

- 第三章：分布式篇 - 复制和分区, 我变复杂了、但也可靠了
- 第四章：控制节点篇 - 数据节点的管理、路由与迁移修复  
- 第五章：元数据篇 - 元数据服务与垃圾回收 (GC)
- 第六章：协议篇 - S3 协议, 对象存储的事实标准
- 第七章：容灾篇 - 容灾与跨区异步复制
- 番外篇：CDN - 其实我也是存储节点