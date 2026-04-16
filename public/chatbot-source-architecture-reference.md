# Chatbot 源码架构参考手册（面向后续改造）

> 这份文档不是“代码审计报告”，而是**源码级架构设计说明**：
> - 8 个核心模块分别怎么设计；
> - 每个模块具备哪些能力；
> - 改需求时应该改哪里、会影响哪里。

---

## 0. 总览：系统是怎么拼起来的

这个项目本质是一个 **Chat Runtime + Artifact Workspace** 双核心系统：

1. **Chat Runtime（对话运行时）**
   - 负责消息收发、流式返回、工具调用（tool use）、会话持久化。
2. **Artifact Workspace（右侧产物工作区）**
   - 负责文档/代码/表格等“可编辑产物”的创建、增量更新、版本管理。

运行时主链路：

- 前端 `useActiveChat` 组织请求 → `/api/chat`
- 服务端做鉴权、限流、模型能力判断、工具编排
- 模型流（text/tool/reasoning/data part）回到前端
- `DataStreamHandler` 把流增量分发到 Chat UI + Artifact 状态机
- 最终结果持久化到 Postgres

---

## 1. Auth 认证系统（模块 01）

## 1.1 架构角色

- 提供统一身份层（`guest` / `regular`）。
- 给后续所有模块注入稳定 `session.user.id`，用于数据归属和权限控制。

## 1.2 设计要点

- NextAuth + Credentials 双 provider：
  - `credentials`：邮箱密码登录。
  - `guest`：自动建库访客账号登录。
- JWT/Session 都带 `id` + `type`，下游模块无需二次查 user。
- Guest 是真实 DB 用户（不是纯前端匿名态），因此 chat/document 的数据模型完全统一。

## 1.3 能力清单

- 注册、登录、访客登录。
- 权限边界：
  - chat/document 查询与写入都按 userId 校验。
- 重定向安全控制（guest 登录回跳路径限制）。

## 1.4 后续改造建议

- 想加组织/团队权限：从 `session.user` 扩展 `orgId/role`，再把 chat/document 查询改为 `(orgId, userId)` 组合策略。

---

## 2. Chat 核心与流式传输（模块 02）

## 2.1 架构角色

- 系统的“会话大脑”：
  - 接收用户输入；
  - 选择模型；
  - 处理工具调用；
  - 输出流式消息；
  - 持久化消息。

## 2.2 请求模型

`/api/chat` 支持两种输入模式：

1. **普通发言**：`message`（最后一条用户消息）
2. **工具审批续流**：`messages`（包含 approval state 的全量上下文）

这让“工具需要用户同意”场景可以自然续跑，不用新建额外 API。

## 2.3 流式执行模型

- 使用 `streamText()` + `createUIMessageStream()`。
- 根据模型能力动态启用/关闭工具。
- 工具包括天气、文档创建、文档更新、建议生成、精确编辑。

## 2.4 持久化策略（关键）

- 用户消息先落库。
- assistant/tool 结果在 `onFinish` 后定稿写库。

这保证了：
- 中途中断至少保留用户输入；
- 完整响应后再固化 assistant 状态。

## 2.5 能力清单

- 模型选择（含能力探测）。
- 流式输出（text/reasoning/tool/data）。
- 工具编排与审批续流。
- 标题异步生成。
- 限流（IP + 用户配额）。

---

## 3. Shell 布局与导航（模块 03）

## 3.1 架构角色

- 提供“应用壳”：侧边栏、主聊天区域、右侧 artifact 区域。
- 路由用于状态表达（`/` 新会话、`/chat/[id]` 已有会话）。

## 3.2 设计特点

- `(chat)/layout.tsx` 是真正 UI 主体。
- `page.tsx` 返回 `null`，页面内容由 shell 常驻管理。
- Sidebar 历史使用 SWR Infinite，支持分页与按日期分组。

## 3.3 能力清单

- 新建会话、切换会话。
- 删除单会话/全部会话（optimistic UI）。
- 可折叠 sidebar 与移动端适配。

## 3.4 改造定位

- 改导航体验：`components/chat/app-sidebar.tsx` / `sidebar-history.tsx`
- 改 shell 布局：`components/chat/shell.tsx` / `app/(chat)/layout.tsx`

---

## 4. Message 消息系统（模块 04）

## 4.1 架构角色

- 负责“消息如何展示”以及“流式期间滚动如何表现”。

## 4.2 渲染分层

- `messages.tsx`：消息列表容器（含空态、滚动到底按钮）。
- `message.tsx`：单条消息 part 渲染器：
  - user text
  - assistant text / reasoning
  - tool states（input / approval / denied / output）
  - file 附件

## 4.3 滚动策略

- `MutationObserver + ResizeObserver` 双驱动。
- 只在“用户处于底部且未主动滚动”时自动跟随。

## 4.4 能力清单

- 流式消息平滑展示。
- reasoning 聚合展示。
- tool 审批按钮（Allow / Deny）。
- 消息操作（编辑、投票等在 action 区）。

---

## 5. Input 输入系统（模块 05）

## 5.1 架构角色

- 负责文本输入、模型切换、附件上传、slash 命令。

## 5.2 现状设计

当前存在两层输入体系：

1. **业务输入层**：`multimodal-input.tsx`
   - 真正用于 chat 页面；
   - 附件上传到 `/api/files/upload` 后再发送 URL。
2. **通用输入层**：`ai-elements/prompt-input.tsx`
   - 组件库式输入能力（Provider、local file 管理、blob/dataURL 处理）。

## 5.3 能力清单

- 文本提问 / 回车提交。
- 模型选择与 cookie 持久化。
- 图片附件上传（JPEG/PNG, <=5MB）。
- 粘贴图片自动上传。
- slash 命令（new/clear/model/theme/delete/purge）。

## 5.4 改造建议

- 中期应统一输入主干，避免双轨长期并存导致维护成本上升。

---

## 6. Artifact 产物系统（模块 06）

## 6.1 架构角色

- 聊天之外的“可编辑结果面板”。
- 由工具流驱动（create/update/edit document）。

## 6.2 状态模型

- 核心状态由 `useArtifact` 管理（SWR 本地 key-value 方式）。
- 字段包括：`documentId/title/kind/content/status/isVisible/boundingBox`。

## 6.3 流驱动协议

data stream part（例如 `data-id`, `data-kind`, `data-textDelta`, `data-finish`）
→ `DataStreamHandler`
→ 对应 artifact definition (`artifacts/*/client.tsx`)
→ 更新右侧面板。

## 6.4 能力清单

- text/code/sheet 产物生成。
- 版本浏览（prev/next/diff）。
- 文本建议（suggestions）。
- 手动编辑与自动保存。

## 6.5 版本语义

- LLM 更新：新增版本（`saveDocument`）。
- 手工编辑：更新当前最新版本内容（`updateDocumentContent`）。

---

## 7. Database & AI Provider（模块 07）

## 7.1 DB 架构

- Drizzle + Postgres。
- 主要实体：User / Chat / Message / Vote / Document / Suggestion / Stream。

## 7.2 数据关系

- Chat 归属 User。
- Message 归属 Chat。
- Vote 关联 Chat + Message。
- Document 采用 `(id, createdAt)` 复合主键做版本化。
- Suggestion 关联到特定 Document 版本。

## 7.3 错误模型

- `ChatbotError = type + surface` 双维度编码。
- 既统一 HTTP status，也控制错误暴露策略（返回前端或仅日志）。

## 7.4 Provider 架构

- 生产环境：Vercel AI Gateway。
- 测试环境：mock provider。
- 动态模型能力探测：tools / vision / reasoning。

## 7.5 能力清单

- 多模型接入。
- 基于能力的工具开关。
- 统一错误语义。

---

## 8. Hooks 与状态管理（模块 08）

## 8.1 架构角色

- 用 hooks 把“聊天运行时状态”按领域分层组织。

## 8.2 中枢：`useActiveChat`

它负责：
- chatId 与 URL 绑定；
- useChat transport 配置；
- data stream 对接；
- 首屏消息装载与会话切换重置；
- model cookie 恢复；
- auto resume；
- votes 拉取。

## 8.3 关键防错设计

- `currentModelIdRef` 解决闭包读取旧模型问题（stale closure）。

## 8.4 能力清单

- 单页聊天状态统一管理。
- 跨路由会话切换的稳定恢复。
- 流式过程中的状态一致性。

---

## 9. 后续改需求：改哪里（快速索引）

## 9.1 改“模型/工具策略”

- `app/(chat)/api/chat/route.ts`
- `lib/ai/models.ts`
- `lib/ai/providers.ts`
- `lib/ai/tools/*`

## 9.2 改“消息渲染体验”

- `components/chat/messages.tsx`
- `components/chat/message.tsx`
- `hooks/use-scroll-to-bottom.tsx`

## 9.3 改“输入与附件体验”

- `components/chat/multimodal-input.tsx`
- `app/(chat)/api/files/upload/route.ts`
- `components/ai-elements/prompt-input.tsx`

## 9.4 改“Artifact 编辑体验”

- `components/chat/artifact.tsx`
- `artifacts/*/client.tsx`
- `artifacts/*/server.ts`
- `app/(chat)/api/document/route.ts`

## 9.5 改“权限与账号体系”

- `app/(auth)/auth.ts`
- `app/(auth)/api/auth/guest/route.ts`
- `lib/db/queries.ts`

---

## 10. 给后续开发的建议使用方式

建议把这份文档当成“改动前检查单”：

1. 先定位模块边界（你要改的是输入、消息、流、还是 artifact）。
2. 按“快速索引”找到主文件和旁路影响文件。
3. 判断是否涉及：
   - 鉴权边界；
   - 持久化语义（新版本还是覆盖）；
   - 流式状态同步（DataStream part）；
   - UI optimistic 行为回滚。

这样能显著降低“改一个点，炸多个模块”的概率。

---

## 11. 一句话版本

这个 chatbot 的源码架构是：

**以 `useActiveChat + /api/chat` 为运行时中枢，以 DataStream 协议连接消息 UI 与 Artifact 工作区，以 Postgres 版本化文档模型承接可编辑产物。**

