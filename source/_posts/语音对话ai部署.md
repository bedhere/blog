---
title: 语音对话ai部署
date: 2026-02-04 02:07:08
updated: 2026-03-30 00:00:00   # 更新日期（可选）
description: 基于硅基流动 API 实现 ASR + LLM + TTS 的全链路语音对话 AI，Spring Boot 后端 + Vue 前端
keywords: [AI, 语音对话，硅基流动，Spring Boot, Vue, ASR, TTS]
tags:
  - AI
  - Spring Boot
  - Vue
categories:
  - [技术，AI]
cover: 
top_img: 
sticky: 

---

## 项目概述

本项目在原有 Spring Boot + Vue 项目基础上，接入 **硅基流动（SiliconFlow）** 平台的 API，实现了完整的语音对话 AI 功能，涵盖以下三个核心环节：

- **ASR（自动语音识别）**：利用浏览器原生 Web Speech API 将用户语音转为文字
- **LLM（大语言模型）**：调用硅基流动的对话接口，生成 AI 回复
- **TTS（文本转语音）**：调用硅基流动的 CosyVoice2-0.5B 模型，将 AI 回复合成为语音播放

整体架构如下：

```
用户说话 → 浏览器 ASR → 文本 → Spring Boot /ai/chat → 硅基流动 LLM → AI 回复文本
                                                                         ↓
用户收听 ← 浏览器 AudioContext 播放 ← Spring Boot /ai/tts → 硅基流动 TTS（CosyVoice2）
```

---

## 技术栈

| 模块 | 技术 |
|------|------|
| 后端框架 | Spring Boot 3.x |
| 前端框架 | Vue 3 + Element Plus |
| AI 对话 | 硅基流动 LLM API |
| 语音合成 | 硅基流动 TTS（FunAudioLLM/CosyVoice2-0.5B） |
| 语音识别 | 浏览器 Web Speech API（SpeechRecognition） |
| HTTP 客户端 | Spring RestTemplate / Axios |

---

## 后端实现

### 1. AI 对话接口（AIController）

`/ai/chat` 接口接收前端传来的用户消息和历史对话记录，调用 `AIService` 向硅基流动 LLM 发起请求，返回 AI 回复文本。

请求体结构：

```json
{
  "message": "你好，今天天气怎么样？",
  "history": [
    { "role": "user", "content": "你是谁？" },
    { "role": "assistant", "content": "我是AI助手。" }
  ]
}
```

响应结构：

```json
{
  "code": "200",
  "data": {
    "reply": "今天天气不错，阳光明媚！"
  }
}
```

- `history` 字段为可选项，传入最近若干条对话记录，使模型具备上下文记忆能力
- 若消息为空则直接返回 400 错误，避免无效调用

---

### 2. TTS 语音合成接口（TTSController）

`/ai/tts` 接口接收文本参数，调用硅基流动 `/v1/audio/speech` 接口，将文本合成为 MP3 音频字节流返回给前端。

关键配置：

- **模型**：`FunAudioLLM/CosyVoice2-0.5B`
- **语音角色**：`FunAudioLLM/CosyVoice2-0.5B:claire`
- **API Key**：通过 `application.yml` 中 `siliconflow.api-key` 注入，不硬编码在代码中

`application.yml` 配置示例：

```yaml
siliconflow:
  api-key: sk-xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

请求方式为 `POST /ai/tts?text=你好`，后端使用 `@RequestParam("text")` 接收，返回 `application/octet-stream` 音频字节流。

---

## 前端实现

### 1. 语音识别（ASR）

前端使用浏览器原生 `SpeechRecognition` API 实现语音输入：

```javascript
const SpeechRecognition = window.SpeechRecognition || window.webkitSpeechRecognition
recognition = new SpeechRecognition()
recognition.lang = 'zh-CN'       // 中文识别
recognition.continuous = false    // 单次识别模式
recognition.interimResults = false // 不返回中间结果
```

- 点击麦克风按钮启动/停止识别
- 识别成功后自动调用 `handleUserMessage()` 发送消息
- 若浏览器不支持或未授权麦克风，界面给出相应提示

> **注意**：Web Speech API 要求运行在 HTTPS 或 `localhost` 环境下，HTTP 部署时需特别注意。

---

### 2. AI 对话请求

识别到文字（或手动输入文字）后，调用后端 `/ai/chat` 接口：

```javascript
const response = await request.post('/ai/chat', {
  message: text,
  history: history  // 最近 10 条历史消息
})
```

- 每次请求携带最近 **10 条**对话历史，保证上下文连贯
- 收到 AI 回复后将消息追加到对话列表，并触发 TTS 播报

---

### 3. 语音播报（TTS）

AI 回复完成后，自动调用 `speakText()` 向后端 `/ai/tts` 请求音频，通过 `AudioContext` 解码并播放：

```javascript
const res = await axios.post(
  "http://localhost:9090/ai/tts",
  params,                        // URLSearchParams 格式
  { responseType: "arraybuffer" } // 接收二进制音频流
)
const audioCtx = new AudioContext()
const audioBuffer = await audioCtx.decodeAudioData(res.data)
const source = audioCtx.createBufferSource()
source.buffer = audioBuffer
source.connect(audioCtx.destination)
source.start()
```

- 使用 `URLSearchParams` 传参，与后端 `@RequestParam` 匹配
- `responseType: "arraybuffer"` 接收二进制音频数据
- 通过 `AudioContext.decodeAudioData()` 解码后直接播放，无需生成临时 audio 标签

---

### 4. 界面功能

| 功能 | 说明 |
|------|------|
| 文字输入 | 输入框回车或点击发送按钮均可触发 |
| 语音输入 | 点击麦克风按钮开始/停止录音，支持中文识别 |
| AI 回复 | 实时显示 AI 回复，并自动语音播报 |
| 消息记录 | 展示完整对话历史，含发送时间 |
| 清空对话 | 点击清空按钮重置对话记录 |
| 加载状态 | 等待 AI 回复期间显示打字动画 |

---

## 完整调用流程

1. 用户点击麦克风按钮，浏览器调起 `SpeechRecognition` 开始录音
2. 识别完成后得到中文文本，显示在对话框中
3. 前端携带文本及历史记录，请求后端 `POST /ai/chat`
4. 后端将消息转发至硅基流动 LLM，获取 AI 回复文本
5. 前端收到回复，展示文字消息
6. 前端自动请求后端 `POST /ai/tts?text=AI回复内容`
7. 后端调用硅基流动 CosyVoice2 TTS，将文本合成为音频字节流返回
8. 前端用 `AudioContext` 解码音频并播放，完成语音播报

---

## 注意事项

- 语音识别依赖浏览器支持，推荐使用 **Chrome 或 Edge** 浏览器
- 需在 **HTTPS 或 localhost** 环境下才可使用麦克风
- 硅基流动 API Key 请妥善保管，不要提交到公开代码仓库
- TTS 接口返回的是原始音频字节流，前端需用 `ArrayBuffer` 方式接收，不可当作 JSON 解析
