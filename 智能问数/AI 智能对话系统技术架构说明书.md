


> 我们要做什么什么样？产品的定位是什么？
> 开源的软件中哪些是适合与我们这个需求的？用这些开源软件之后，我们还需要做什么？
> 技术栈的选择应该怎么选？
> 整体的jx'g




版本: v1.0
> 状态: 规划中
> 适用场景: 构建支持流式对话、多模型切换及“思考模型”可视化的高性能聊天应用。

---
## 1. 总体架构设计 (System Architecture)

本项目采用 **前后端分离 (Decoupled)** 的架构模式。前端专注于交互体验与流式渲染，后端作为核心逻辑层，负责模型调度、上下文管理及数据持久化。

### 1.1 核心分层

- **接入层 (Client Layer):** 基于 Web 的用户界面，负责用户输入捕获、Markdown 渲染、流式响应解析。
- **接口层 (API Gateway):** 统一的 HTTP/WebSocket 接口，负责鉴权、限流与请求分发。
- **业务层 (Service Layer):** 处理对话上下文、提示词工程 (Prompt Engineering) 及业务逻辑（如搜索、工具调用）。
- **模型层 (Model Layer):** 通过适配器统一接入 DeepSeek, OpenAI, Claude 等大模型。
- **数据层 (Data Layer):** 存储会话历史、用户配置及向量数据（可选）。

---

## 2. 技术栈选型 (Technology Stack)

### 2.1 前端 (Frontend)

| **模块**          | **技术组件**                     | **选型理由**                                                       |
| --------------- | ---------------------------- | -------------------------------------------------------------- |
| **核心框架**        | **Next.js 14+ (App Router)** | 提供 SSR/CSR 混合渲染，优秀的流式数据 (Streaming) 支持。                        |
| **开发语言**        | **TypeScript**               | 强类型约束，保证前后端接口数据结构的一致性。                                         |
| **UI 组件库**      | **Shadcn/UI + Tailwind CSS** | 高度可定制的现代化组件，轻量且符合 ChatGPT 风格。                                  |
| **状态管理**        | **React Hooks / Zustand**    | 处理流式数据的实时追加与 UI 状态同步。                                          |
| **Markdown 渲染** | **React Markdown**           | 支持 LaTeX 公式 (`remark-math`)、代码高亮 (`react-syntax-highlighter`)。 |
| **HTTP 客户端**    | **Native Fetch / Axios**     | 配合 `TextDecoder` 处理 Server-Sent Events (SSE)。                  |

### 2.2 后端 (Backend)

| **模块**     | **技术组件**          | **选型理由**                         |
| ---------- | ----------------- | -------------------------------- |
| **核心框架**   | **Python 3.10+**  | AI 领域的原生语言，生态最丰富。                |
| **Web 框架** | **FastAPI**       | 高性能异步框架，原生支持 ASGI 和流式响应，并发处理能力强。 |
| **数据验证**   | **Pydantic**      | 严格的数据模型定义与验证，自动生成 OpenAPI 文档。    |
| **应用服务器**  | **Uvicorn**       | 基于 uvloop 的高性能 ASGI 服务器。         |
| **环境管理**   | **Poetry / Venv** | 依赖包管理。                           |

### 2.3 AI 适配与编排 (AI Orchestration)

|**模块**|**技术组件**|**说明**|
|---|---|---|
|**模型网关**|**LiteLLM**|**核心组件**。统一 OpenAI、DeepSeek、Claude 等 100+ 模型的 API 调用格式，作为“防腐层”。|
|**高级编排**|**LangChain** (可选/后期)|用于构建 Agent、RAG（检索增强生成）或复杂任务链。|

### 2.4 数据持久化 (Storage)

|**模块**|**技术组件**|**说明**|
|---|---|---|
|**关系型数据库**|**PostgreSQL**|存储 User, Session, Message 等结构化数据。|
|**ORM**|**SQLAlchemy / Prisma**|Python 端的数据操作层。|

---

## 3. 详细功能组件设计

### 3.1 前端核心组件 (Frontend Components)

1. **`ChatInterface` (主控组件):**
    - 管理消息列表 (`messages[]`) 状态。
    - 处理 `StreamReader`，将后端返回的二进制流解码为文本。
    - 实现“打字机”滚动逻辑。
2. **`MessageRenderer` (消息渲染器):**
    - 区分 `user` 和 `assistant` 样式。
    - 集成 `Remark` 插件链，支持：
        - 表格渲染
        - LaTeX 数学公式 ($E=mc^2$)
        - Mermaid 流程图 (可选)
3. **`ThinkingBubble` (思维链可视化):**
    - **功能:** 专门用于解析推理模型（如 DeepSeek-R1）返回的 `<think>...</think>` 标签。
    - **交互:** 默认折叠，点击可展开查看 AI 的推理过程，将“思考”与“回答”视觉分离。

### 3.2 后端模块设计 (Backend Modules)

1. **路由层 (`/api/chat`):**
    - 接收标准 OpenAI 格式的 JSON 请求。
    - 验证 API Key 或用户 Token。
    - 返回 `StreamingResponse` (MIME: `text/event-stream`)。
2. **服务层 (`ChatService`):**
    - **上下文截断:** 计算 Token 数，确保不超出模型限制（Window Slicing）。
    - **System Prompt 注入:** 动态插入“角色设定”或“当前时间”等元数据。
3. **适配层 (`LLMAdapter`):**
    - 基于 `LiteLLM` 封装。
    - 实现统一的错误处理（如 Key 过期、额度不足的重试机制）。

---

## 4. 关键数据流 (Data Flow)

### 4.1 流式对话交互流程

1. **用户动作:** 用户在前端输入问题，点击发送。
2. **请求封装:** 前端将当前问题 + 最近 N 轮历史记录打包为 JSON。
3. **后端处理:**
    - FastAPI 接收请求，通过 Pydantic 校验数据格式。
    - `ChatService` 调用 `LiteLLM`，向模型提供商发起流式请求 (`stream=True`)。
4. **流式转发:**
    - LiteLLM 收到模型的一个 Chunk (碎片)。
    - FastAPI 立即将该 Chunk 写入 HTTP 响应流。
5. **前端解析:**
    - 前端 `fetch` 读取流。
    - 检测到 `<think>` 标签，更新 `isThinking` 状态，渲染到折叠面板。
    - 检测到普通文本，追加到当前消息气泡。

---

## 5. 扩展性与数据治理规划 (Scalability & Governance)

_针对后续可能的数据治理及多应用接入需求预留的接口：_

1. **元数据管理 (Metadata):**
    - 在消息表中预留 `meta_info` (JSONB) 字段，用于存储该条回复引用的“数据来源”、“置信度”或“耗时”。
2. **多 Agent 路由:**
    - 后端可扩展 `Router` 模块，根据用户意图（如“写SQL”、“查血缘”），将请求分发给不同的 Python Agent 处理。
3. **内容审计:**
    - 在流式返回前增加“敏感词过滤”中间件 (Middleware)。
