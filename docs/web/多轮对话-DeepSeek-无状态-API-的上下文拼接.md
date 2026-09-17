---
title: 多轮对话：DeepSeek 无状态 API 的上下文拼接
url: https://api-docs.deepseek.com/zh-cn/guides/multi_round_chat
source_type: web
author: null
tags:
- DeepSeek
- 多轮对话
- 无状态API
- 上下文管理
- OpenAI SDK
summary: DeepSeek 对话 API 无状态，多轮对话需在每次请求时把完整历史 messages 传给服务端，否则模型丢失上下文。
fetched_at: '2026-09-17T06:42:47.432524+00:00'
---

DeepSeek `/chat/completions` API 是一个“无状态” API，即服务端不记录用户请求的上下文，用户在每次请求时，**需将之前所有对话历史拼接好后**，传递给对话 API。

`from openai import OpenAIclient = OpenAI(api_key="<DeepSeek API Key>", base_url="https://api.deepseek.com")# Round 1messages = [{"role": "user", "content": "What's the highest mountain in the world?"}]response = client.chat.completions.create(    model="deepseek-flash",    messages=messages)messages.append(response.choices[0].message)print(f"Messages Round 1: {messages}")# Round 2messages.append({"role": "user", "content": "What is the second?"})response = client.chat.completions.create(    model="deepseek-flash",    messages=messages)messages.append(response.choices[0].message)print(f"Messages Round 2: {messages}")`