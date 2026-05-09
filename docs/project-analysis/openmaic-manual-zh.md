# OpenMAIC 项目手册（代码分析版）

> 基于仓库当前代码结构与 README 的 Project Structure 进行整理，覆盖：整体功能、模块职责、调用链路、后端接口、AI 模型服务配置方式。

## 1. 项目功能总览

OpenMAIC 是一个基于 Next.js App Router 的 AI 互动课堂系统：用户输入主题或资料后，系统通过多阶段生成流程产出课程大纲、场景内容（幻灯片/测验/互动/PBL）、多智能体课堂讨论、TTS 语音讲解、图像/视频生成及课堂回放。前端 UI 与后端 API 在同一仓库中实现（BFF 形态）。

核心能力：
- 课程生成：大纲流式生成 + 场景内容补全。
- 课堂互动：多智能体聊天、白板动作、测验评分、PBL 对话。
- 多模态：图像生成、视频生成、TTS、ASR、PDF 解析。
- 课堂管理：课堂保存/读取、异步课堂任务、服务健康检查。

## 2. 模块与目录职责（按 Project Structure + 代码目录）

### 2.1 `app/`（应用层）
- `app/page.tsx`：首页，课程生成入口。
- `app/classroom/[id]/`：课堂播放页。
- `app/api/`：服务端 API 路由集合（见第 4 节完整接口文档）。

### 2.2 `lib/`（核心逻辑层）
- `lib/generation/`：两阶段课程生成（大纲 -> 场景）。
- `lib/orchestration/`：多智能体编排（LangGraph/状态流）。
- `lib/playback/`：课堂播放状态机（播放控制）。
- `lib/action/`：课堂动作执行（语音、白板、聚光等）。
- `lib/ai/`：大模型抽象（模型元数据、provider、thinking config）。
- `lib/audio/`：TTS/ASR provider 抽象与适配。
- `lib/media/`：图像/视频 provider 抽象与适配。
- `lib/pdf/`：PDF 解析 provider。
- `lib/server/provider-config.ts`：服务端 provider 统一配置入口（读取 env/yaml）。

### 2.3 `components/`（展示层）
- 幻灯片编辑渲染、课堂聊天、设置面板、白板、智能体 UI 等。

### 2.4 `packages/`（工作区包）
- `pptxgenjs`：PPT 导出相关定制。
- `mathml2omml`：公式转换能力。

## 3. 调用方式与典型链路

## 3.1 本地启动
1. 安装依赖：`pnpm install`
2. 环境准备：`cp .env.example .env.local`
3. 运行开发：`pnpm dev`

## 3.2 课程生成主链路（典型）
1. 前端调用 `POST /api/generate/scene-outlines-stream` 生成大纲（流式）。
2. 针对每个场景调用 `POST /api/generate/scene-content` 生成内容。
3. 若启用多模态，调用 `POST /api/generate/image` / `POST /api/generate/video`。
4. 若启用旁白，调用 `POST /api/generate/tts`。
5. 保存课堂：`POST /api/classroom`；读取课堂：`GET /api/classroom?id=...`。

## 3.3 实时互动链路（典型）
- 多智能体讨论：`POST /api/chat`（SSE/流式响应）。
- PBL 场景对话：`POST /api/pbl/chat`。
- 测验评分：`POST /api/quiz-grade`。
- 语音转写：`POST /api/transcription`（ASR）。

## 4. 后端接口文档（`app/api/**`）

> 注：以下为路由级接口清单；字段按代码中显式读取项整理。

### 4.1 访问控制与系统

| 方法 | 路径 | 作用 | 主要输入 |
|---|---|---|---|
| GET | `/api/access-code/status` | 查询 ACCESS_CODE 鉴权状态 | Cookie `openmaic_access` |
| POST | `/api/access-code/verify` | 提交访问码并写入访问状态 | JSON body |
| GET | `/api/health` | 健康检查 | 无 |

### 4.2 课堂与任务

| 方法 | 路径 | 作用 | 主要输入 |
|---|---|---|---|
| POST | `/api/classroom` | 保存/创建课堂数据 | JSON body |
| GET | `/api/classroom?id=...` | 读取课堂数据 | Query: `id` |
| POST | `/api/generate-classroom` | 提交异步课堂生成任务 | JSON body |
| GET | `/api/generate-classroom/{jobId}` | 查询任务状态/结果 | Path: `jobId` |

### 4.3 课程生成（核心）

| 方法 | 路径 | 作用 | 主要输入 |
|---|---|---|---|
| POST | `/api/generate/scene-outlines-stream` | 流式生成课程大纲 | JSON body；Header 可含 `x-image-generation-enabled` `x-video-generation-enabled` |
| POST | `/api/generate/scene-content` | 生成场景内容 | JSON body |
| POST | `/api/generate/scene-actions` | 生成场景动作脚本 | JSON body |
| POST | `/api/generate/agent-profiles` | 生成/补全智能体设定 | JSON body |

### 4.4 多模态媒体

| 方法 | 路径 | 作用 | 主要输入 |
|---|---|---|---|
| POST | `/api/generate/image` | 生成图片 | JSON body（ImageGenerationOptions）；Header: `x-image-provider` `x-image-model` `x-api-key` `x-base-url` |
| POST | `/api/generate/video` | 生成视频 | JSON body（VideoGenerationOptions）；Header: `x-video-provider` `x-video-model` `x-api-key` `x-base-url` |
| POST | `/api/proxy-media` | 代理下载媒体资源 | JSON body: `url` |

### 4.5 语音与文本

| 方法 | 路径 | 作用 | 主要输入 |
|---|---|---|---|
| POST | `/api/generate/tts` | 文本转语音（TTS） | JSON body |
| POST | `/api/transcription` | 语音转文本（ASR） | `multipart/form-data`：`audio`、`providerId`、`modelId`、`language`、`apiKey`、`baseUrl` |
| POST | `/api/azure-voices` | 获取 Azure 语音列表/能力 | JSON body |

### 4.6 教学互动能力

| 方法 | 路径 | 作用 | 主要输入 |
|---|---|---|---|
| POST | `/api/chat` | 多智能体课堂聊天（流式） | JSON body |
| POST | `/api/pbl/chat` | PBL 场景对话 | JSON body |
| POST | `/api/quiz-grade` | 自动评分 | JSON body |

### 4.7 外部能力与 provider 校验

| 方法 | 路径 | 作用 | 主要输入 |
|---|---|---|---|
| POST | `/api/parse-pdf` | PDF 解析（可切 provider） | `multipart/form-data`：`pdf`、`providerId`、`apiKey`、`baseUrl` |
| POST | `/api/web-search` | 联网搜索 | JSON body |
| POST | `/api/verify-model` | 校验聊天模型可用性 | JSON body |
| POST | `/api/verify-image-provider` | 校验图像 provider | Header: `x-image-provider` `x-image-model` `x-api-key` `x-base-url` |
| POST | `/api/verify-video-provider` | 校验视频 provider | Header: `x-video-provider` `x-video-model` `x-api-key` `x-base-url` |
| POST | `/api/verify-pdf-provider` | 校验 PDF provider | JSON body |
| GET | `/api/server-providers` | 读取服务端 provider 配置摘要 | 无 |

## 5. 第三方 AI 模型服务配置方式（chat / image / video / tts / asr）

## 5.1 配置优先级（推荐理解）
1. `.env.local`：统一放密钥、Base URL、默认模型。
2. `server-providers.yml`：可配置服务端 provider（适合部署集中管理）。
3. 请求级覆盖：部分接口支持通过 Header（如 image/video）或 form-data（如 ASR/PDF）传入临时 provider 参数。

## 5.2 Chat（LLM）配置
- 关键变量：
  - 各厂商 API Key（如 `OPENAI_API_KEY`、`ANTHROPIC_API_KEY`、`GOOGLE_API_KEY`、`GLM_API_KEY`、`MINIMAX_API_KEY` 等）。
  - `DEFAULT_MODEL`：服务端默认聊天模型（格式常见 `provider:model`）。
- 代码位置：`lib/ai/providers.ts`、`lib/ai/llm.ts`、`lib/server/provider-config.ts`。

## 5.3 Image 配置
- 接口：`POST /api/generate/image`
- 支持方式：
  - 环境变量中配置 `IMAGE_<PROVIDER>_API_KEY` / `IMAGE_<PROVIDER>_BASE_URL`。
  - 请求头覆盖：`x-image-provider`、`x-image-model`、`x-api-key`、`x-base-url`。
- provider 适配集合：`lib/media/image-providers.ts` + `lib/media/adapters/*image*`。

## 5.4 Video 配置
- 接口：`POST /api/generate/video`
- 支持方式：
  - 环境变量中配置 `VIDEO_<PROVIDER>_API_KEY` / `VIDEO_<PROVIDER>_BASE_URL`。
  - 请求头覆盖：`x-video-provider`、`x-video-model`、`x-api-key`、`x-base-url`。
- provider 适配集合：`lib/media/video-providers.ts` + `lib/media/adapters/*video*`。

## 5.5 TTS 配置
- 接口：`POST /api/generate/tts`
- 支持方式：
  - 环境变量配置（如 `TTS_<PROVIDER>_API_KEY`、`TTS_<PROVIDER>_BASE_URL`）。
  - 特殊 provider（如 VoxCPM）可仅配置 Base URL。
- provider 代码：`lib/audio/tts-providers.ts`、`lib/audio/voxcpm.ts`。

## 5.6 ASR 配置
- 接口：`POST /api/transcription`
- 支持方式：
  - 环境变量配置（如 `ASR_<PROVIDER>_API_KEY`、`ASR_<PROVIDER>_BASE_URL`）。
  - 请求级 form-data 可传 `providerId`、`modelId`、`apiKey`、`baseUrl`、`language`。
- provider 代码：`lib/audio/asr-providers.ts`。

## 5.7 配置示例（最小可运行）

```env
# Chat
OPENAI_API_KEY=sk-...
DEFAULT_MODEL=openai:gpt-5.5

# Image
IMAGE_OPENAI_API_KEY=sk-...
IMAGE_OPENAI_BASE_URL=https://api.openai.com/v1

# Video
VIDEO_MINIMAX_API_KEY=...
VIDEO_MINIMAX_BASE_URL=https://api.minimaxi.com

# TTS
TTS_MINIMAX_API_KEY=...
TTS_MINIMAX_BASE_URL=https://api.minimaxi.com

# ASR
ASR_LEMONADE_BASE_URL=http://localhost:13305/v1
```

## 6. 建议的二次开发入口

1. 新增聊天模型：从 `lib/ai/providers.ts` 与 `lib/ai/model-metadata.ts` 入手。
2. 新增图片/视频模型：在 `lib/media/adapters/` 新增 adapter，并注册到 provider 列表。
3. 新增 TTS/ASR：在 `lib/audio/*-providers.ts` 增加 provider 定义与调用逻辑。
4. 新增接口：在 `app/api/<feature>/route.ts` 增加路由，并复用 `lib/` 的 provider 抽象层。

