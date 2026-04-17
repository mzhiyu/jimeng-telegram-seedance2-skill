This is a sanitized public workflow skill for Dreamina / Seedance video generation automation.

The repository documents a production-oriented workflow skill that connects:

**WSL -> Windows -> Dreamina CLI -> Telegram delivery**

This is not just a prompt file or a simple code snippet.  
It is a structured workflow document based on real execution experience, covering parameter confirmation, asset preprocessing, path conversion, task submission, status polling, result download, and final delivery.

## What this project does

This skill is designed to help standardize and automate the following process:

- parse user generation requests
- prepare and confirm submission previews
- preprocess local image, audio, and video assets
- convert paths between WSL and Windows environments
- call Dreamina / Seedance CLI commands
- poll task status
- download generated results
- send final videos through Telegram

## Prerequisites (Important)

This repository is **not a plug-and-play tool**.

Before using this skill, you must already have:

- a properly installed and configured **Dreamina CLI / Jimeng CLI**
- your **own authenticated account** logged in through the CLI
- a working local runtime environment, including WSL / Windows path handling, allowed directories, and required dependencies

In other words, this repository shares the **workflow skill document**, not the Dreamina CLI itself, and does not include any account credentials, login state, or secrets.

## Why this project matters

The value of this project lies in:

- end-to-end workflow design
- cross-environment automation
- practical production constraints and safeguards
- reusable skill-style documentation
- delivery-oriented process design

## Repository contents

- `SKILL.md` — main workflow skill document
- `README.md` — project overview

## Notes

- This is a **sanitized public version**
- Private usernames, paths, file names, submit IDs, and URLs have been replaced
- This repository is shared for workflow demonstration, learning, and portfolio purposes

## Status

Sanitized and published  
Based on real workflow experience  
Public portfolio version

## License

MIT

----------------------------------------------------------------------------------------------------

中文说明 / Chinese
这是一个经过脱敏处理的公开版工作流 skill，用于说明如何将 Dreamina / Seedance 的视频生成流程整理成可复用的自动化执行规范。

该项目展示的是一个面向实际生产流程的 skill 文档，用于串联：
**WSL -> Windows -> Dreamina CLI -> Telegram 交付**

它不是单纯的提示词文件，也不是单纯的脚本片段，而是一份基于真实使用经验整理出来的完整工作流规范，包含参数确认、素材预处理、路径转换、任务提交、状态轮询、结果下载和最终交付等步骤。

## 这个项目能做什么

这个 skill 用于规范和自动化以下流程：

- 解析用户的视频生成指令
- 生成提交前预览单并确认参数
- 预处理本地图片、音频、视频素材
- 处理 WSL 与 Windows 之间的路径转换
- 调用 Dreamina / Seedance CLI 提交任务
- 轮询任务状态
- 下载生成结果
- 通过 Telegram 发送最终视频

## 使用前提（非常重要）

本仓库 **不是开箱即用工具**。  
在使用这个 skill 之前，必须先满足以下前提条件：

- 你已经在本地正确安装并配置好 **Dreamina / 即梦 CLI**
- 你已经使用**自己的账号**完成 CLI 登录
- 你已经具备可正常调用的本地运行环境（如 WSL / Windows 路径、白名单目录、依赖工具等）

也就是说，这个仓库公开的是 **workflow skill 文档**，不是 Dreamina CLI 本体，也不包含任何账号、密钥或可直接代替登录的信息。

## 项目价值

这个项目的重点不在算法本身，而在于：

- 端到端工作流设计
- 跨环境自动化整合
- 真实生产问题的规避与约束
- 可复用的技能文档化表达
- 面向交付的流程规范

## 仓库内容

- `SKILL.md`：主 skill 文档
- `README.md`：项目说明

## 说明

- 本仓库为**公开脱敏版本**
- 已替换私人用户名、路径、文件名、submit_id、URL 等信息
- 本仓库用于工作流展示、学习交流与作品集展示用途

## 状态

已完成脱敏整理  
已公开发布  
基于真实流程经验整理

## License

MIT
