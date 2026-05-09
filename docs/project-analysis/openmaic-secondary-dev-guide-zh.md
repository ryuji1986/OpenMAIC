# OpenMAIC 二次开发指导手册（用户系统 + 前后端分离）

> 目标：在不直接给出代码实现的前提下，明确 **要改哪些模块、改到什么程度、按什么顺序推进**。本手册针对两个改造目标：
> 1) 新增用户模块（注册/登录）；
> 2) 将当前 `app/api/**` 后端能力迁移到独立服务，OpenMAIC 主项目改为纯前端。

---

## 0. 改造范围与原则

## 0.1 当前架构现状（起点）
- 当前项目是 Next.js App Router，`app/api/**` 同仓实现 BFF/后端路由。
- 前端页面与后端路由共用部分 `lib/**` 能力（生成、provider、媒体、语音、存储等）。
- 访问控制仅有 `ACCESS_CODE` 级别，不是用户账号体系。

## 0.2 目标架构（终点）
- **Web 前端仓库（当前仓库）**：只保留页面、交互、状态管理、API Client、鉴权态管理。
- **独立后端服务（新仓库或子仓）**：承载原 `app/api/**` 全部业务能力，新增用户系统与鉴权。
- 前后端通过 HTTPS API 通信，统一鉴权方案（推荐 JWT + Refresh Token）。

## 0.3 改造原则
1. **先兼容、再切换**：先抽象 API Client，再逐步替换调用，避免一次性重构风险。
2. **接口契约先行**：先定义 OpenAPI/接口文档，再做迁移。
3. **鉴权中台化**：所有业务接口都通过统一鉴权中间件。
4. **能力等价迁移**：先保证行为一致，再做性能/架构优化。

---

## 1. 总体实施路线（建议分 4 期）

## Phase 1：接口契约与前端解耦准备
**产出**：
- 独立 `API Contract`（建议 OpenAPI 3.1）
- 前端统一 API 调用层（`apiClient`）
- 环境变量切换能力（本地/测试/生产后端基地址）

**关键动作**：
1. 盘点现有接口（`app/api/**`）并冻结 v1 契约。
2. 前端所有 `fetch('/api/...')` 改造为 `apiClient.xxx()`。
3. 新增 `NEXT_PUBLIC_API_BASE_URL`（仅指向独立后端）。

## Phase 2：用户系统设计与落地（后端优先）
**产出**：
- 用户表与鉴权表结构
- 注册/登录/刷新/登出/当前用户信息接口
- 前端登录态管理 + 路由守卫

## Phase 3：业务接口迁移（后端独立化）
**产出**：
- 原 `app/api/**` 能力在独立后端全部可用
- 前端所有业务调用改为新后端地址
- 文件上传、流式响应、长任务轮询、provider 校验保持可用

## Phase 4：收口与纯前端化
**产出**：
- 删除或停用 `app/api/**`
- 统一错误码、日志、监控、限流
- 发布迁移说明和运维文档

---

## 2. 功能一：新增用户模块（注册/登录）需要做的修改

## 2.1 后端数据模型（建议）
至少新增以下实体：
1. `users`
   - `id`、`email/phone/username`（唯一）
   - `password_hash`
   - `status`（active/disabled）
   - `created_at/updated_at/last_login_at`
2. `auth_refresh_tokens`（如用 Refresh Token）
   - `id`、`user_id`、`token_hash`、`expires_at`、`revoked_at`
3. 可选：`user_profiles`、`user_settings`（个性化配置）

## 2.2 鉴权方案（建议）
- **Access Token（短期）+ Refresh Token（长期）**。
- Access Token 放 `Authorization: Bearer`。
- Refresh Token 放 HttpOnly Cookie（优先）或安全存储。
- 后端统一鉴权中间件校验 token 并注入 `currentUser`。

## 2.3 认证接口（最小集）
1. `POST /auth/register`
2. `POST /auth/login`
3. `POST /auth/refresh`
4. `POST /auth/logout`
5. `GET /auth/me`

建议补充：
- `POST /auth/forgot-password`
- `POST /auth/reset-password`
- `POST /auth/verify-email`

## 2.4 前端需要修改的模块
1. **页面与路由**
   - 新增登录页、注册页、（可选）找回密码页。
2. **状态管理**
   - 新增 `auth store`：保存用户信息、access token 生命周期状态。
3. **请求拦截**
   - `apiClient` 自动注入 token；401 自动触发 refresh（带并发保护）。
4. **路由守卫**
   - 课堂编辑、生成、历史记录等页面改为登录可见。
5. **用户关联数据**
   - 课堂数据增加 `user_id` 归属，前端列表按“我的课堂”查询。

## 2.5 安全与合规注意事项
- 密码必须 hash（如 Argon2/Bcrypt），禁止明文。
- 注册/登录接口增加限流、验证码（防撞库）。
- CORS 仅允许前端域名；开启 HTTPS。
- 敏感日志脱敏（token、apiKey、手机号邮箱）。

---

## 3. 功能二：后端独立服务化（本项目纯前端）修改清单

## 3.1 现有后端能力迁移清单
将 `app/api/**` 路由迁移到独立后端，分组如下：
1. 系统与访问控制：`health`、`access-code/*`
2. 课堂与任务：`classroom`、`generate-classroom/*`
3. 生成链路：`generate/*`
4. 教学互动：`chat`、`pbl/chat`、`quiz-grade`
5. 媒体与语音：`generate/image`、`generate/video`、`generate/tts`、`transcription`
6. 工具能力：`parse-pdf`、`web-search`、`proxy-media`
7. provider 校验：`verify-*`、`server-providers`

## 3.2 前端仓库改造点（必须）
1. 删除服务端路由依赖
   - 不再依赖 `app/api/**`，统一走 `NEXT_PUBLIC_API_BASE_URL`。
2. 建立 API SDK 层
   - 按领域拆分：`authApi`、`classroomApi`、`generationApi`、`mediaApi`、`providerApi`。
3. 统一错误模型
   - 后端返回 `{ code, message, details, requestId }`，前端统一 toast/重试。
4. 流式与上传适配
   - `chat`/`scene-outlines-stream` 保持 SSE/stream 消费能力。
   - `parse-pdf`/`transcription` 保留 `multipart/form-data` 上传。
5. 环境配置
   - `.env.local` 迁移出后端私密变量；前端只保留 `NEXT_PUBLIC_*`。

## 3.3 后端服务改造点（必须）
1. 架构建议
   - 可选 Node.js（NestJS/Fastify/Express）或 Python（FastAPI）。
   - 统一中间件：日志、鉴权、限流、异常、请求 ID。
2. provider 配置管理
   - 由后端托管第三方密钥（OpenAI/视频/TTS/ASR等），前端不再持有 secret。
3. 异步任务
   - `generate-classroom` 等长任务建议引入任务队列（Redis + queue）。
4. 存储层
   - 将课堂、用户、任务状态落库（PostgreSQL 推荐）。
5. 可观测性
   - 接口耗时、失败率、第三方 provider 调用成功率监控。

## 3.4 CORS / Cookie / 域名策略
- 前端域名：`web.example.com`
- 后端域名：`api.example.com`
- 若使用 Cookie 刷新 token：需正确设置 `SameSite=None; Secure`。
- CORS 白名单严格限制到前端域。

---

## 4. 接口契约重构建议（从“页面直连路由”到“领域 API”）

将原路由按业务重新分层为：
1. `Auth API`：注册、登录、刷新、登出、当前用户
2. `Classroom API`：创建、查询、列表、删除、任务状态
3. `Generation API`：大纲、内容、动作、agent profiles
4. `Interaction API`：chat、pbl、quiz-grade
5. `Media API`：image/video/tts/asr/proxy
6. `Provider API`：server-providers、verify-model、verify-image/video/pdf

每个 API 都应定义：
- 请求参数 schema
- 响应 schema
- 错误码
- 鉴权要求（公开/登录/管理员）
- 限流策略

---

## 5. 迁移执行清单（可直接作为项目管理任务）

## 5.1 前端任务单
- [ ] 建立 `src/services/apiClient`（超时、重试、401 刷新、requestId）
- [ ] 新增 `src/services/authApi` 与登录态 store
- [ ] 新增登录/注册页面与表单校验
- [ ] 全量替换现有 API 调用到 SDK
- [ ] 为流式接口补充断线重连与取消逻辑
- [ ] 清理对 `app/api/**` 的调用

## 5.2 后端任务单
- [ ] 初始化独立服务脚手架 + CI
- [ ] 落地用户/鉴权数据模型
- [ ] 实现 auth 五件套接口
- [ ] 迁移 classroom / generation / media / interaction 接口
- [ ] 对接第三方 provider 与密钥托管
- [ ] 接入数据库、缓存、队列、对象存储
- [ ] 补充 OpenAPI 文档与 Postman 集合

## 5.3 测试与验收任务单
- [ ] 鉴权流程：注册->登录->刷新->登出->失效
- [ ] 业务流程：生成课堂->播放->互动->保存->读取
- [ ] 多模态流程：image/video/tts/asr 全链路
- [ ] 兼容性：超时、断网、provider 失败、重试
- [ ] 安全性：未登录拦截、越权访问、限流生效

---

## 6. 风险点与应对

1. **流式接口迁移风险**：SSE 在网关层可能被缓冲。
   - 应对：网关关闭缓冲，增加心跳包与前端超时重连。
2. **第三方 provider 不稳定**：模型响应超时/失败。
   - 应对：provider fallback + 统一重试与熔断。
3. **前端暴露密钥风险**：若未彻底分离会泄露 secret。
   - 应对：所有 secret 仅后端保存，前端仅传业务参数。
4. **历史数据迁移风险**：课堂数据结构变化导致读取失败。
   - 应对：做数据迁移脚本与版本字段兼容。

---

## 7. 交付物清单（你最终应拿到）

1. 《前后端分离 API 契约文档》（OpenAPI）
2. 《用户系统设计说明》（数据模型 + 鉴权流程图）
3. 《迁移实施计划》（里程碑 + 负责人 + 时间）
4. 《联调与验收用例》
5. 《上线回滚预案》

---

## 8. 建议的实施顺序（一句话版）

**先抽 API 客户端并定义契约，再做后端用户系统，再迁业务接口，最后删除 `app/api/**` 完成纯前端化。**
