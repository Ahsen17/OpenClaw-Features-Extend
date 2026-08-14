<div align="center">

# 🦞 OpenClaw Features Extend

### Extend OpenClaw with real-time voice interaction capabilities

<p>
Voice Input · Speech Recognition · Text-to-Speech
</p>

<p>
<a href="./README_CN.md">🇨🇳 中文文档</a>
</p>

<br/>

[![OpenClaw Extension](https://img.shields.io/badge/OpenClaw-Extension-orange?style=flat-square)](https://github.com/openclaw/openclaw)
[![Qwen ASR](https://img.shields.io/badge/Qwen-Realtime%20ASR-blue?style=flat-square)](https://dashscope.aliyun.com/)
[![Chrome TTS](https://img.shields.io/badge/TTS-Chrome%20Native-green?style=flat-square)](https://www.google.com/chrome/)

</div>


---

## ✨ Overview

**OpenClaw Features Extend** is an extension project designed to enhance the interaction capabilities of [OpenClaw](https://github.com/openclaw/openclaw).

The project currently focuses on adding **real-time voice interaction**, enabling OpenClaw to support both voice input and voice output.

Current capabilities:

- 🎙️ **Real-time Speech Recognition**
  - Integrates Alibaba Cloud **DashScope Qwen Realtime ASR**
  - Converts microphone audio into text in real time

- 🔊 **Real-time Speech Output**
  - Uses Chrome native **Text-to-Speech (TTS)**
  - Converts OpenClaw responses into natural voice output


The goal:

> Make OpenClaw interaction more natural, moving from text-based communication toward voice-first AI experiences.


---

# 🚀 Features


## 🎙️ Real-time Speech Recognition


Powered by **DashScope Qwen Realtime ASR**, enabling real-time voice input for OpenClaw.


```mermaid
flowchart LR

    A[🎙️ Microphone Input]
        --> B[🔊 Audio Stream]

    B --> C[☁️ DashScope API]

    C --> D[🤖 Qwen Realtime ASR]

    D --> E[📝 Text Output]

    E --> F[🦞 OpenClaw]
