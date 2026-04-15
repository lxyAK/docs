# Banana Slides 源码深度分析

> 📚 系统性的 Banana Slides 源码架构分析文档
>
> 分析版本：v0.4.0 | 分析日期：2026-04-15
> 仓库：https://github.com/Anionex/banana-slides

---

## 📖 文档索引

| 序号 | 文档 | 内容概览 |
|------|------|---------|
| 00 | [总体架构概览](./00-总体架构概览.md) | 项目定位、目录结构、数据模型、核心数据流、API 设计 |

---

## 🎯 快速上手指南

### 阅读路径

```
00-总体架构概览 → 了解全局
```

---

## 🍌 项目简介

Banana Slides 是一个 AI 原生 PPT 生成应用，基于 nano banana pro 模型，支持：
- 一句话/大纲/描述 → 自动生成 PPT
- 上传模板图片，AI 遵循风格
- 可编辑 PPTX 导出（OCR + Inpainting）
- 多 AI Provider 热切换
- CLI 批量生成工具
