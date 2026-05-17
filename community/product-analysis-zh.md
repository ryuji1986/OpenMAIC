# OpenMAIC 课堂生成流程调研（从 `/generation-preview` 出发）

> 调研日期：2026-05-17  
> 入口范围：`app/generation-preview/page.tsx` 及其调用链（前端 + API + 生成引擎）

---

## 0. 先说结论：这是一个“预览页触发 + 分阶段生成 + 进课堂续跑”的架构

从 `generation-preview` 进入后，系统不是一次把整门课全做完，而是：

1. 在预览页先做“准备工作”（PDF、检索、大纲、智能体、首个场景内容/动作/TTS）。
2. 首场景成功后立刻跳转到 `/classroom/[id]`，把“剩余场景”交给课堂页的生成器继续跑。
3. 媒体（图/视频）是并行启动、异步落库的辅助通道，不阻塞主链路。

这种设计的产品目标是**尽快首帧可见**（先让用户进入课堂），而不是“全部生成完成再展示”。

---

## 1. 宏观流程梳理（不讲细节，只讲每段职责）

### Step A：页面启动与会话恢复（`/generation-preview`）
- 组件加载后先从 `sessionStorage` 读 `generationSession`，恢复上次状态（包括是否在大纲审阅态）。
- 若判断需要自动开始，就调用 `startGeneration()`。

### Step B：可选 PDF 解析
- 如果 session 里还有 `pdfStorageKey` 且未有 `pdfText`，则调用 `/api/parse-pdf`。
- 产物包括文本与图片元数据；文本会截断到上限；图片会写入 IndexedDB，供后续视觉提示使用。

### Step C：可选联网检索
- 如果需求开启 web search，就请求 `/api/web-search`。
- 返回 `context + sources`，把 `researchContext` 注入后续大纲生成。

### Step D：大纲流式生成（SSE）
- 调 `/api/generate/scene-outlines-stream`，边收边显示 outline。
- SSE 事件类型包括：`languageDirective`、`outline`、`retry`、`done`、`error`。
- 完成后进入“自动继续”或“人工审阅大纲”分支。

### Step E：可选智能体人设生成
- 当设置为 `agentMode=auto` 时，调用 `/api/generate/agent-profiles`。
- 生成教师/同学人设（含头像/语音），写入本地 registry 与 stage 绑定。

### Step F：预览页只生成“第一个场景”
- 先调 `/api/generate/scene-content` 生成首场景内容；
- 再调 `/api/generate/scene-actions` 生成首场景动作脚本；
- 若启用 TTS，再调 `/api/generate/tts` 为首场景 speech 动作产音频。

### Step G：进入课堂页继续生成剩余场景
- 首场景写入 store 后，跳转 `/classroom/{stageId}`。
- 同时把 `generationParams` 放入 `sessionStorage`，课堂页据此恢复上下文继续生成剩余场景。

### Step H：课堂页剩余场景生成器（串行主链）
- `useSceneGenerator` 对 pending outlines 做“逐场景串行”：内容 -> 动作 -> TTS -> 写场景。
- 任一场景失败会暂停并标记 failed，可单独重试。
- 媒体生成由 `generateMediaForOutlines` 在旁路执行。

---

## 2. 按流程详细讲解（逐函数 / 逐段逻辑）

## 2.1 `/generation-preview` 页生命周期与自动启动

### 关键函数/状态
- `GenerationPreviewContent`：预览页主组件。
- `persistSession`：统一更新 React state + `sessionStorage`。
- `startGeneration`：主流程编排器。
- `waitForOutlineReviewChoice`：控制“大纲审阅 vs 自动继续”。

### 逻辑要点
1. `useEffect` 初始化时读取 `generationSession`，兼容老状态，恢复 `previewPhase`。  
2. 如果 phase 属于 `preparing / generating-content / (review但无outlines)`，自动触发 `startGeneration()`。  
3. 整个生成 run 绑定一个 `AbortController`，离开页面时统一 abort，避免悬挂请求。

---

## 2.2 Step A 详细：PDF 解析链路（可选）

### 调用
- `fetch('/api/parse-pdf', { method: 'POST', body: FormData })`

### 输入
- PDF 文件本体。
- 可选 providerId/apiKey/baseUrl（支持不同 PDF 解析后端）。

### 输出处理
1. `parseResult.data.text` 写入 `pdfText`（并按 `MAX_PDF_CONTENT_CHARS` 截断）。
2. 抽取 `metadata.pdfImages`（或兼容旧 `images` 数组）。
3. `storeImages(images)` 把图片存 IndexedDB，返回 `imageStorageIds`。
4. 把轻量 `pdfImages`（含 storageId）回写 session。
5. 若文本/图片超限，写 `truncationWarnings` 供 UI 告警。

### 职责定位
- 这一步的本质是把重型输入（PDF）归一成后续可消费的上下文：`pdfText + pdfImages + imageMapping`。

---

## 2.3 Step B 详细：Web Search（可选）

### 调用
- `fetch('/api/web-search', ...)`

### 输入
- 查询词（需求文本）、可选 `pdfText`、provider 配置。

### 输出处理
1. UI 保存 `sources` 用于来源展示。
2. `searchData.context` 写入 `researchContext`，后续注入大纲 prompt。

### 职责定位
- 给大纲生成增加“时效/外部事实”补充上下文，尤其适合新闻、趋势、人物等主题。

---

## 2.4 Step C 详细：流式大纲生成（核心）

### 调用
- `fetch('/api/generate/scene-outlines-stream', ...)` + 手动读取 SSE `ReadableStream`。

### 事件协议
- `languageDirective`：先给“授课语言指令”。
- `outline`：逐条场景大纲增量返回。
- `retry`：服务端触发重试时清空并重播。
- `done`：结束，携带最终 outlines。
- `error`：失败终止。

### 客户端处理策略
1. 增量更新 `streamingOutlines`，用户可边看边确认方向。
2. 流完成后根据 `reviewOutlineEnabled` 和“中途展开编辑器意图”决定：
   - 进入人工审阅；或
   - 2.5 秒后自动继续。
3. 最终把 `sceneOutlines + languageDirective` 持久化到 session。

### 设计价值
- 这是“感知性能”优化关键：用户先看到结构，再继续重任务。

---

## 2.5 Step D 详细：智能体自动生成（可选）

### 调用
- `fetch('/api/generate/agent-profiles', ...)`

### 输入
- `stageInfo`、简化场景摘要、`languageDirective`。
- 可用头像池、头像描述、可用语音列表。

### 输出处理
1. 把 agents 持久化到本地 agent registry（`saveGeneratedAgents`）。
2. stage 绑定 `agentIds`。
3. 通过 reveal modal 让用户“看卡片”确认，再继续后续生成。
4. 若失败则回退到 preset agents。

### 职责定位
- 解决“课堂角色感”和语音匹配问题，属于体验增强层，不是生成主干的硬依赖。

---

## 2.6 Step E 详细：预览页只做首场景（首帧加速）

### 内容生成
- `POST /api/generate/scene-content`
- 输入：`outline + allOutlines + pdfImages + imageMapping + stageInfo + agents + languageDirective`

### 动作生成
- `POST /api/generate/scene-actions`
- 输入：`(effectiveOutline|outline) + content + stageId + agents + previousSpeeches + userProfile + languageDirective`

### 首场景 TTS（可选）
- 对首场景 `speech` actions 串行调用 `/api/generate/tts`。
- 成功后把 base64 转 Blob，写 `db.audioFiles`。
- 只要有失败且存在 speech，即整体抛错中断。

### 最终落地
1. `store.addScene(data.scene)` + `setCurrentSceneId`。
2. 其余 outlines 放入 `generatingOutlines` 作为占位。
3. 把 `generationParams`（pdfImages/agents/userProfile/languageDirective）写入 sessionStorage。
4. `router.push('/classroom/{stageId}')`。

---

## 2.7 Step F 详细：课堂页接力生成剩余场景

### 接力点
- `app/classroom/[id]/page.tsx` 会读取 preview 页写入的 `generationParams`（注释已说明）。

### 主执行器
- `useSceneGenerator().generateRemaining(params)`。

### 执行逻辑
1. 计算 pending outlines（按 order 排序，排除已完成场景）。
2. 启动媒体旁路：`generateMediaForOutlines(...)`。
3. 主链路串行 for-loop：
   - `fetchSceneContent` -> `fetchSceneActions` -> `generateTTSForScene` -> `addScene`
4. 失败处理：
   - 任何一步失败 -> `addFailedOutline` + status `paused`。
5. 支持 `retrySingleOutline(outlineId)` 从头补跑该场景。

### 职责定位
- 这是“完整课堂最终收敛”的主引擎；预览页只是点火器。

---

## 2.8 服务端接口族与职责分层

## `/api/generate/scene-content`
- 单场景“内容层”生成（slide elements / quiz questions / interactive html / pbl config）。

## `/api/generate/scene-actions`
- 单场景“行为层”生成（讲解、对白、交互动作序列）。

## `/api/generate/scene-outlines-stream`
- 课程级结构层（SSE 流式）。

## `/api/generate/tts`
- 语音层，把动作文本转为可播放音频。

## `/api/generate/image`、`/api/generate/video`
- 媒体资产层，异步旁路。

## `/api/generate-classroom` 与 `/api/generate-classroom/[jobId]`
- 这是另一套“任务化后端作业”入口：创建 job + 轮询状态，不是当前 `/generation-preview` 主流程直接调用的链路。

---

## 3. 流程图（文字版）

1. Home 填需求 -> push `/generation-preview`。  
2. Preview 恢复 session，自动 `startGeneration`。  
3. (可选) parse PDF -> 存文本/图。  
4. (可选) web-search -> researchContext。  
5. SSE 大纲生成 -> (可选)用户审阅确认。  
6. (可选)自动生成 agents。  
7. 生成首场景 content/actions/(TTS)。  
8. 首场景入库，跳 `/classroom/{id}`。  
9. Classroom 用 `useSceneGenerator` 串行补齐其余场景；媒体并行旁路持续产出。  
10. 全部完成或失败暂停等待重试。

---

## 4. 你最关心的“慢”通常来自哪几段（结合流程定位）

1. **首场景链路中的顺序调用**：`content -> actions -> (N次TTS)` 串行，且 TTS 失败会中断。  
2. **课堂页剩余场景主链路串行**：每个 outline 都是完整两步 + 可选 TTS，累积时延显著。  
3. **媒体任务内部串行**：虽然是旁路，但大量视频会拉长“全部就绪”时间。  
4. **大纲审阅等待窗口**：默认可能进入人工确认或 2.5 秒自动继续，属于产品上可感知等待。

---

## 5. 术语速记（便于二次开发）

- `generationSession`：预览页会话（sessionStorage）。
- `generationParams`：预览页跳课堂时传递的继续生成参数（sessionStorage）。
- `sceneOutlines`：课程结构骨架。
- `languageDirective`：全流程语言一致性控制信号。
- `generatingOutlines`：课堂中“待生成骨架”占位集合。


---

## 6. 前端视角：动态课堂内容 / Scene Runtime 是怎么展示出来的？

这一层可以理解为：**生成引擎负责“产出 scene + actions”，Scene Runtime 负责“按时间和状态把 scene 演出来”**。

### 6.1 运行时主入口（Classroom 页面）

- 课堂页入口是 `app/classroom/[id]/page.tsx`。
- 页面先从 IndexedDB / 服务端恢复 `stage + scenes`，再判断是否有未完成 outlines：
  - 有未完成：调用 `useSceneGenerator.generateRemaining(...)` 边学边生成；
  - 已完成：至少恢复/续跑媒体任务（图视频）。
- 然后渲染 `<Stage />`，真正的 Scene Runtime 就在这个组件内。

**你可以把这层理解为：**
- `ClassroomPage` = 数据恢复 + 续跑调度器
- `Stage` = 课堂播放器内核（渲染 + 播放 + 互动）

### 6.2 Runtime 状态中枢：Zustand Store + Stage API

Scene Runtime 并不是直接操作 DOM，而是靠“状态驱动渲染”：

1. `useStageStore` 存放核心课堂状态：`stage / scenes / currentSceneId / generatingOutlines / failedOutlines`。
2. `Stage API`（`lib/api/stage-api*.ts`）提供一组“场景操作语义”：
   - `scene`（增删改查场景）
   - `navigation`（next/prev/goTo）
   - `element`（slide 元素增删改）
   - `canvas`（highlight/spotlight/zoom/laser 等视觉效果）
3. 运行时动作执行后，只改 store；React 组件订阅 store 自动重渲。

这使得 Scene Runtime 具备“可回放、可中断、可恢复”的一致行为。

### 6.3 Scene 渲染分发：按类型挂不同 Renderer

`components/stage/scene-renderer.tsx` 是类型分发器：

- `slide` -> `SlideEditor`（作为播放态渲染器使用）
- `quiz` -> `QuizView`
- `interactive` -> `InteractiveRenderer`
- `pbl` -> `PBLRenderer`

也就是说“动态课堂”不是一个单一画布，而是**多场景类型统一挂载协议**。

### 6.4 Slide Runtime：不是静态图，而是可被动作驱动的画布

在 slide 场景下，运行时会把当前 scene 的 elements/background/theme 喂给 `slide-renderer`。
随后动作系统会动态施加：

- 元素显隐/强调
- spotlight / laser / zoom
- 白板层叠加
- 语音同步高亮

因此用户看到的是“随讲解推进而变化”的画面，而不是一次性渲染完毕的静态 PPT。

### 6.5 时间轴与动作执行：PlaybackEngine + ActionEngine

`Stage` 组件内部关键对象：

- `PlaybackEngine`：控制播放状态机（idle / playing / discussion 等）、节奏和触发事件。
- `ActionEngine`：解释执行 scene.actions（speech / highlight / discuss / wait ...），把动作翻译成对 store/API 的状态变更。
- `audioPlayer`：处理讲解音频播放。

典型过程：
1. 进入场景，初始化该场景动作序列。
2. `ActionEngine` 按序推进动作。
3. 每步动作改变 store 或播放音频。
4. React UI 基于 store 变化刷新（画面、头像状态、气泡、特效）。

### 6.6 讲授层 + 讨论层并行：Roundtable / Chat / TTS

Runtime 不只有“老师播片”，还支持实时讨论：

- `Roundtable` 显示多智能体发言状态、音频指示器。
- `ChatArea` 管理会话流，支持继续/暂停/终止讨论。
- `useDiscussionTTS` 在讨论模式下为不同 agent 生成或播放语音。
- `Stage` 里维护 `lectureSpeech`（讲授）与 `liveSpeech`（讨论）两套文本流状态。

这让课堂可在“脚本讲授”与“实时互动”之间切换。

### 6.7 边生成边展示（Incremental Runtime）

动态感的另一个来源是**数据增量到达**：

1. 首场景先进入课堂并可立即播放。
2. `useSceneGenerator` 在后台继续生成后续 scene。
3. 每生成一个 scene 就 `addScene(scene)`，侧边栏与导航立刻可见。
4. 如果失败，outline 被标记为 failed，可单独重试，不阻塞已可播放部分。

所以用户体验上是“课程在成长”，不是“加载完成后一次性出现”。

### 6.8 媒体资源异步水合（Hydration）

图像/视频生成由 `media-orchestrator` 在旁路执行：

- 任务状态放在 `useMediaGenerationStore`
- 结果 blob 放 IndexedDB
- 完成后写 object URL 回任务状态

Scene Renderer 读取到可用 URL 后自然替换占位，形成“素材后到、画面自动变清晰/可播”的效果。

### 6.9 Scene Runtime 的核心抽象（一句话）

**Scene Runtime = Store（真相源） + Action Timeline（驱动） + Typed Renderers（呈现） + Async Hydration（后到资源补全）**。

