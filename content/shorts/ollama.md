---
title: "Running local LLMs with Ollama"
date: 2026-01-30
draft: false
tags: ["AI", "Ollama", "LLM"]
---

In your terminal, run: ollama run gpt-oss:20b

Inside the Ollama prompt, type: >>> /set parameter num_ctx 32768 >>> /save gpt-oss-32k

Exit and restart Claude Code with the new model name: claude --model gpt-oss-32k

