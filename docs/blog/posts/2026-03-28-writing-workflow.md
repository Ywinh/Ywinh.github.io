---
date: 2026-03-28
categories:
  - 写作
tags:
  - Workflow
  - Writing
---

# 中文博客写作流程

推荐把每篇文章都放在 `docs/blog/posts/` 下，并通过文件名加日期保持可读性。

<!-- more -->

一个简单可持续的流程是：

1. 新建一篇带日期的 Markdown 文件。
2. 在 YAML 头部补齐 `date`、`categories`、`tags`。
3. 在正文前半部分加入摘要分隔符 `<!-- more -->`。
4. 启动 `mkdocs serve` 本地预览。

这样首页摘要、分类页和标签页都会更稳定。
