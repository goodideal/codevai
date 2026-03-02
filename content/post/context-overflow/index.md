---
title: "Luna 不说话的那个下午"
date: 2026-02-15T00:00:00+08:00
slug: "context-overflow"
author: Jerry
image: cover.svg
categories:
    - 技术
tags:
    - OpenClaw
    - CoDevAI
    - 踩坑
draft: false
---

那天下午，Luna 突然不说话了。

我刚让她跑了几个后台扫尾任务，她很快回来确认：`data_sync.py` 已经在 iMac 后台稳定运行，每分钟准时更新实盘数据面板。一切正常。

然后我补了句需求——想把网站通过 Tailscale 暴露成内部服务，手机上也能看。

问了句「怎么样？」

没有回应。

等了一会儿，又没有。我开始怀疑：**死机了？**

---

## /status 才是答案

下意识敲了几个命令，`/help`，`/status`。

`/status` 的输出把事情说清楚了：

```
Context: 1.0m / 400k (256%)
Compactions: 1
LLM request rejected: Your input exceeds the context window of this model
```

翻译成人话：这个 session 里堆了大约 100 万 token，但当前模型最多处理 40 万，超了 2.56 倍，模型直接拒绝了请求。

它不是没看到我的消息。它是连进入推理阶段的资格都没有。

---

## 我做了个典型的错误动作

看到没反应，我的第一反应是：**是不是模型不行？**

于是我把模型从 `gemini-3-flash-preview` 换成了 `openai-codex/gpt-5.2-codex`，再次 `/status`：

```
Context: 1.0m / 400k (256%)
```

一模一样。

这一刻我才彻底明白：**换模型不会清空 session 的历史上下文。** Context 爆的是 session，不是模型。在一个已经超载的 session 里，换谁来算，结局都一样。

状态里还有一行：`Compactions: 1`——系统已经帮我自动压缩过一次了，还是撑不住。`/compact` 是止痛药，不是时光倒流。100 万 token 的 session，压缩救不了。

---

## 怎么走到这一步的

回头看那条对话，我在同一个 session 里做了：

- 后台任务确认
- 网络暴露方案讨论
- 使用体验确认
- 多次 `/status`、`/help`
- 反复贴日志输出

每件事单独看都不大。但 Context 只会单向膨胀，没有退路。

我把一个 session 当成了「万能对话框」，这是根本原因。

---

## 正确的止血方式

再遇到这类情况，正确的顺序是：

1. 直接 `/new` 开新 session
2. 或者 `/reset` 彻底清空
3. 想保留指令风格但丢掉历史：`/compact instructions`

记住两件事：`/model` 只能换模型，救不了 Context；`/status` 只能诊断，不能修复。

---

## 现在我怎么用 OpenClaw

**一个 session，只做一件事。**

我现在把 session 当一次性工单：一个 session 只处理 `data_sync` 的运行状态，一个 session 专门讨论 Tailscale 方案，一个 session 只用来设计访问策略。任务结束，session 也结束。

**阶段性做 Snapshot。**

有了结论就立刻让 Luna 整理一份摘要：当前目标、已完成的事、关键配置、未解决的问题、下一步。我保存的是这份 Snapshot，而不是完整的聊天记录。新 session 开头只贴 Snapshot，干净、高效。

**少做「看起来没事」的动作。**

反复 `/status`、把长日志来回贴、在同一个 session 里不断换话题——这些都是 Context 的隐形杀手。Codex 写长文章也是。

---

如果那天重来，我会这样做：后台任务确认完就结束 session，新开一个专门讨论 Tailscale，得出方案后立刻 Snapshot，再开下一个处理手机访问和安全策略。

整个过程会更干净，也更可控。

---

这次经历让我想清楚一件事：

**OpenClaw 的 session 是易失的，结论才是资产。**

只要还把 session 当「无限聊天记录」用，Context 爆炸就只是时间问题。学会关 session，学会保存 Snapshot，OpenClaw 才会真正变成一个稳定、可长期使用的工具。

---

*— Jerry，CoDevAI 创始人*
