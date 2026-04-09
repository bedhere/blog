---
title: langchain-chatchat项目部署
date: 2026-01-30 02:07:08
updated: 2026-03-30 00:00:00   # 更新日期（可选）
description: 基于 GitHub LangChain-Chatchat 项目 + Xinference 本地 AI 部署的完整流程
keywords: [LangChain-Chatchat, Xinference, 本地 AI, RAG, 知识库问答]
tags:
  - AI
  - LangChain
  - 本地部署
categories:
  - [技术，AI]
cover: 
top_img: 
sticky: 

---

## 项目概述

本文记录我在本地部署 **GitHub LangChain-Chatchat** 项目的完整流程。该项目是一个基于 LangChain 的知识库问答系统，支持本地文档上传、向量检索、RAG（检索增强生成）等功能。我使用 **Xinference** 部署本地大模型，实现了完全离线的智能问答服务。

整体架构如下：

```
用户提问 → Chatchat 前端 → Chatchat 后端 → 意图识别 → 工具调用/RAG 检索
                                              ↓
                                         Xinference 本地 AI（Qwen1.5-Chat / Qwen3-Embedding）
```

---

## 项目目录结构

```
python-ai/
├── chatchat-app/              # Chatchat 主应用
│   ├── data/
│   │   ├── knowledge_base/    # 知识库文件存放目录
│   │   ├── logs/              # 日志目录
│   │   ├── media/             # 多媒体文件
│   │   ├── ntlk_data/         # NLTK 数据
│   │   └── temp/              # 临时文件
│   ├── venv/                  # Chatchat 虚拟环境
│   ├── basic_settings.yaml    # 基础配置文件
│   ├── kb_settings.yaml       # 知识库配置
│   ├── model_settings.yaml    # 模型配置
│   ├── prompt_settings.yaml   # Prompt 模板配置
│   └── tool_settings.yaml     # 工具配置
│
├── xinference-server/         # Xinference 服务端
│   └── venv/                  # Xinference 虚拟环境
│
└── .venv/                     # 全局虚拟环境（可选）
```

---

## 环境准备

### 1. Python 版本要求

- 推荐使用 **Python 3.10+**
- 为 Chatchat 和 Xinference 分别创建独立的虚拟环境，避免依赖冲突

```bash
# 创建 Chatchat 虚拟环境
cd chatchat-app
python -m venv venv
venv\Scripts\activate

# 创建 Xinference 虚拟环境
cd ../xinference-server
python -m venv venv
venv\Scripts\activate
```

---

## Xinference 本地 AI 部署

### 1. 安装 Xinference

```bash
# 激活 Xinference 虚拟环境
xinference-server\venv\Scripts\activate

# 安装 Xinference
pip install "xinference[all]"
```

### 2. 启动 Xinference 服务

```bash
# 启动本地服务（默认端口 9997）
xinference-local --host 127.0.0.1 --port 9997
```

启动后访问 `http://127.0.0.1:9997` 打开 Web 界面。

### 3. 下载并加载模型

在 Xinference Web 界面中搜索并启动以下模型：

- **LLM 模型**：`qwen1.5-chat`（通义千问对话模型）
- **Embedding 模型**：`Qwen3-Embedding-0.6B`（文本向量化模型）

或者使用命令行：

```bash
# 启动 LLM 模型
xinference launch --model-name qwen1.5-chat --model-size 7

# 启动 Embedding 模型
xinference launch --model-name Qwen3-Embedding-0.6B
```

---

## LangChain-Chatchat 配置

### 1. 模型配置（model_settings.yaml）

关键配置项如下：

```yaml
# 默认 LLM 模型
DEFAULT_LLM_MODEL: qwen1.5-chat

# 默认 Embedding 模型
DEFAULT_EMBEDDING_MODEL: Qwen3-Embedding-0.6B

# 历史对话轮数
HISTORY_LEN: 3

# 温度参数（控制回复随机性）
TEMPERATURE: 0.7

# 模型平台配置
MODEL_PLATFORMS:
  - platform_name: xinference
    platform_type: xinference
    api_base_url: http://127.0.0.1:9997/v1
    api_key: EMPTY
    api_concurrencies: 5
    auto_detect_model: false
    llm_models:
      - qwen1.5-chat
    embed_models:
      - Qwen3-Embedding-0.6B
```

### 2. Prompt 模板配置（prompt_settings.yaml）

#### 意图识别模板

```yaml
preprocess_model:
  default: |
    你只要回复 0 和 1，代表不需要使用工具。以下几种问题不需要使用工具:
    1. 需要联网查询的内容
    2. 需要计算的内容
    3. 需要查询实时性的内容
    如果我的输入满足这几种情况，返回 1。其他输入，请你回复 0，你只要返回一个数字
    这是我的问题:
```

该模板用于判断用户问题是否需要调用外部工具或知识库。

#### RAG 问答模板

```yaml
rag:
  default: |
    【指令】根据已知信息，简洁和专业的来回答问题。
    如果无法从中得到答案，请说"根据已知信息无法回答该问题"，
    不允许在答案中添加编造成分，答案请使用中文。

    【已知信息】context

    【问题】question
```

该模板确保 AI 基于检索到的知识库内容作答，避免胡编乱造。

#### 对话历史模板

```yaml
llm_model:
  with_history: |
    The following is a friendly conversation between a human and an AI.
    The AI is talkative and provides lots of specific details from its context.
    If the AI does not know the answer to a question, it truthfully says it does not know.

    Current conversation:
    history
    Human: input
    AI:
```

---

## 启动 Chatchat 服务

### 1. 安装依赖

```bash
# 激活 Chatchat 虚拟环境
chatchat-app\venv\Scripts\activate

# 安装依赖
pip install -r requirements.txt
```

### 2. 初始化知识库

```bash
# 创建知识库目录
mkdir -p data/knowledge_base

# 上传文档到知识库
# 将 PDF、TXT、MD 等格式文件放入 knowledge_base 目录
```

### 3. 启动服务

```bash
# 启动 Chatchat 后端
python server.py

# 启动 Chatchat 前端（新窗口）
python webui.py
```

启动后访问 `http://localhost:8501` 打开 Web 界面。

---

## 功能测试

### 1. 知识库问答

1. 在 Web 界面上传本地文档（支持 TXT、PDF、Word、Markdown 等格式）
2. 点击"向量化"按钮，将文档转换为向量存储
3. 在对话框提问，系统自动检索相关知识并作答

示例问题：
- "项目部署流程是什么？"
- "如何配置 API Key？"
- "支持哪些文件格式？"

### 2. 普通对话

直接提问无需检索的问题，AI 会基于自身知识回答：
- "你好，介绍一下你自己"
- "今天天气怎么样？"

### 3. 多轮对话

系统自动保留最近 **3 轮** 对话历史，支持上下文连贯交流。

---

## 配置优化建议

### 1. 性能调优

- **并发数**：`api_concurrencies: 5` 可根据硬件调整，CPU 较弱时降低此值
- **最大长度**：`MAX_TOKENS` 设为 4096，避免显存溢出
- **流式输出**：`streaming: false` 改为 `true` 可实现打字机效果

### 2. 模型选择

- **对话质量优先**：使用 `qwen1.5-chat` 或 `chatglm3-6b`
- **响应速度优先**：使用更小参数量的模型（如 1.8B、3B）
- **多模态需求**：配置 `image_model` 支持文生图功能

### 3. 知识库管理

- 定期清理无用文档，减少向量库体积
- 对大型 PDF 进行拆分，提高检索精度
- 使用标签分类管理不同主题的知识库

---

## 常见问题

### 1. 模型加载失败

- 检查 Xinference 服务是否正常运行
- 确认 `api_base_url` 配置正确（`http://127.0.0.1:9997/v1`）
- 查看日志文件 `logs/chatchat.log` 排查错误

### 2. 向量化速度慢

- Embedding 模型对 CPU 要求较高，建议使用 GPU 加速
- 批量处理文档时降低并发数
- 考虑使用更小的 Embedding 模型

### 3. 回答质量不佳

- 调整 `temperature` 参数（0.05~0.9），值越低越保守
- 增加 `history_len` 提升上下文理解能力
- 优化 Prompt 模板，添加更多约束条件

---

## 总结

通过以上步骤，我成功部署了本地的 LangChain-Chatchat 知识库问答系统，结合 Xinference 提供的本地 AI 服务，实现了完全离线、安全可控的智能问答功能。该系统适用于企业内部知识库、个人文档管理等场景，无需依赖外部 API，保护数据隐私。