<div align="center">

# 🦞 OpenClaw 功能扩展


### 为 OpenClaw 增加实时语音交互能力


<p>
语音输入 · 实时语音识别 · 实时语音播报
</p>


<p>
<a href="./README.md">🇺🇸 English</a>
</p>


<br/>

[![OpenClaw Extension](https://img.shields.io/badge/OpenClaw-Extension-orange?style=flat-square)](https://github.com/openclaw/openclaw)
[![Qwen ASR](https://img.shields.io/badge/Qwen-Realtime%20ASR-blue?style=flat-square)](https://dashscope.aliyun.com/)
[![Chrome TTS](https://img.shields.io/badge/TTS-Chrome%20Native-green?style=flat-square)](https://www.google.com/chrome/)

</div>


---

# ✨ 项目简介


**OpenClaw Features Extend** 是一个面向 [OpenClaw](https://github.com/openclaw/openclaw) 的功能扩展项目。


项目当前主要增强 OpenClaw 的：

- 🎙️ 实时语音输入能力
- 🔊 实时语音输出能力


让 OpenClaw 从传统文本交互，进一步支持更加自然的人机语音交互。


当前支持：


- 🎙️ **实时语音识别**
  - 基于阿里云 **DashScope Qwen Realtime ASR**
  - 将麦克风输入实时转换为文本


- 🔊 **实时语音播报**
  - 调用 Chrome 原生 TTS
  - 将 OpenClaw 回复转换为语音输出


项目目标：

> 让 AI Agent 的交互方式更加自然。


---

# 🚀 功能介绍


## 🎙️ 实时语音识别


通过 **DashScope Qwen Realtime ASR** 实现实时语音转文本。


```mermaid
flowchart LR


    A[🎙️ 麦克风输入]

        --> B[🔊 音频流]


    B --> C[☁️ DashScope API]


    C --> D[🤖 Qwen Realtime ASR]


    D --> E[📝 文本输出]


    E --> F[🦞 OpenClaw]
