# 小红书 AI Agent

本项目是一个本地运行的内容生产辅助工具，聚焦「AI 资讯 / AI 工具」赛道，帮助完成从选题到草稿的流程。

## 项目目标

- 按关键词检索小红书候选笔记
- 基于互动数据进行 Top N 筛选
- 使用 AI 进行相关性判断与内容改写
- 生成多页图文卡片草稿供人工审核
- 导出可手动发布的内容包（V1 不自动发布）

## 核心特性

- 本地 Web 控制台（配置、任务、审核、导出）
- MCP 接入层（`xiaohongshu-mcp` 适配）
- 结构化 AI 流水线（分类 -> 改写 -> 卡片脚本）
- 本地图文资源生成与草稿管理

## 仓库结构

- `docs/superpowers/specs/`：产品与技术设计文档
- `docs/superpowers/plans/`：分阶段实现计划
- `xiaohongshu-mcp-windows-amd64/`：Windows 下 MCP 工具文件

## 版本说明

- V1 仅提供采集、筛选、改写、生成、审核与导出能力
- V1 不包含自动发布能力

## 相关链接

- GitHub: [13035721381/xiaohongshu-agent](https://github.com/13035721381/xiaohongshu-agent)
