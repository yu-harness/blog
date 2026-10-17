---
title: "{{ replace .File.ContentBaseName \"-\" \" \" | title }}"
summary: ""
date: {{ .Date }}
draft: true

# ── 数字花园字段 ──────────────────────────────────────────────
# series / series_order：系列弧，同系列的文章会在侧边栏排在一起，
#                        在 /series/<name>/ 下生成带进度的系列目录页。
series: ""
series_order: 1

# maturity：这篇笔记的生长状态，会渲染成徽标。
#   seed      🌱 种子：刚丢进来，结论没验证过
#   budding   🌿 生长：骨架成型，数字待复核
#   evergreen 🌳 常青：跑过、复核过，可以放心引用
maturity: seed

# lastReviewed：最后一次复核的日期。首页「最近生长」按它倒序。
lastReviewed: {{ .Date.Format "2006-01-02" }}

# aliases：改版/迁移前的旧 URL，防止外部链接 404。数组形式。
# aliases:
#   - /old/path/

tags: []
---
