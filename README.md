# AI 小说创作平台（ainovel-platform）

> 面向中文网文作者、独立作者与小型内容工作室的 **AI 小说创作工作台**。
> 不做「一句话自动写完一本书」，而是帮助作者**管理长期连载创作**：从选题、大纲、人物与世界观，到章节生成、设定校验、版本管理与导出，最终由作者掌控定稿。

核心差异化：**长篇连续性 · 作者可控 · 成本透明 · 模型可替换**。

技术栈：`Java 17/21` + `Spring Boot 3.4` + `Spring AI 1.0` + `PostgreSQL(pgvector)` + `Redis` + `MinIO` + `Vue 3 + TypeScript + Vite`，`Docker Compose` 一键部署。

---

## 功能特性

| 模块 | 能力 |
| --- | --- |
| 用户与鉴权 | 邮箱注册 / 登录、JWT 无状态令牌、作品级所有权隔离（越权 403/404） |
| 作品管理 | 作品 CRUD、复制（设定+大纲）、归档、删除、额度卡片 |
| 故事策划台 | 雪花写作法五步：核心种子 → 角色动力学 → 角色状态 → 世界观 → 情节架构；每步可锁定 |
| 章节大纲 | 分块生成大纲、悬念节奏曲线（张力可视化）、编辑 / 锁定 / 局部重生成 |
| 章节编辑 | 防抖自动保存、版本号乐观锁（409 冲突保留用户文本）、AI 操作栏、SSE 流式预览与采纳 |
| 设定库 | 人物 / 地点 / 组织 / 规则 / 道具；剧情事实与伏笔状态机（proposed→confirmed/superseded/rejected） |
| 连续性审校 | 问题类型 + 严重度 + 证据（来源章节+摘录）+ 建议 + 置信度；**绝不自动改稿** |
| 灵感罗盘 | 关系图谱（基线 / 章节 delta / 时间轴快照）、5 类逻辑审计、单文件 HTML 导出 |
| 生成任务 | 异步执行、状态机（queued→running→succeeded/failed/cancelled）、Idempotency-Key 幂等、取消、SSE |
| 用量与成本 | 追加式账本（reserve/consume/release）、额度预检、按模型估算费用 |
| 导出 | TXT / Markdown / 打印友好 DOCX / 关系图谱 HTML |
| 工程化 | 统一响应（data+request_id）、全局错误码、OpenAPI 文档、Flyway 迁移、单元测试 |
| 离线模式 | 无模型 Key 也能跑通全流程（内置 Mock 引擎，与真实链路同一套解析路径） |

---

## 目录结构

```
ainovel-platform/
├── backend/                     Spring Boot 3.4 + Spring AI 1.0 后端
│   ├── src/main/java/com/ainovel/
│   │   ├── common/              统一响应 / 错误码 / 全局异常 / request_id
│   │   ├── config/              Security / OpenAPI / 异步 / 应用配置
│   │   ├── security/            JWT 签发校验、当前用户
│   │   ├── modules/             user · project · outline · chapter · storyentity
│   │   │                        · fact · graph · generation · review · export
│   │   │                        · usage · admin
│   │   └── ai/                  AiEngine · PromptRepository · MockAiContentFactory
│   │                            · VectorSyncService · pipeline/* · convert/AiModels
│   ├── src/main/resources/
│   │   ├── prompts/             architecture · chapter · compass · utility 模板(.st)
│   │   └── db/migration/        V1__init.sql · V2__vector_store.sql
│   └── Dockerfile
├── frontend/                    Vue 3 + TS + Vite 前端
│   └── src/                     api/ components/ views/ stores/ utils/ router/
├── deploy/                      docker-compose.yml · .env.example
└── docs/                        功能说明 · 环境搭建 · 微信公众号宣传文 · README
```

---

## 快速开始

### 方式一：Docker Compose 一键起全套

```bash
cd deploy
cp .env.example .env      # 至少修改 JWT_SECRET；要接真实模型再填 LLM_API_KEY
docker compose up -d --build
```

- 前端：http://localhost
- 接口文档：http://localhost:8080/swagger-ui.html
- 健康检查：http://localhost:8080/actuator/health

> 想完全离线体验，在 `.env` 中设置
> `AI_MOCK_ENABLED=true`、`AI_CHAT_PROVIDER=none`、`AI_EMBED_PROVIDER=none`、`AI_VECTORSTORE_TYPE=none`。

### 方式二：本地开发

```bash
# 1) 只起依赖服务
cd deploy && cp .env.example .env && docker compose up -d db redis minio

# 2) 后端（离线演示，无需模型 Key）
cd ../backend
mvn -DskipTests spring-boot:run "-Dspring-boot.run.profiles=mock"

# 3) 前端
cd ../frontend
npm install
npm run dev            # http://localhost:5173（已配置 /api 代理到 8080）
```

### 构建与测试

```bash
cd backend
mvn -B -DskipTests package     # 产出 target/ainovel-server-1.0.0.jar
mvn -B test                    # 25 个纯单元测试（无需数据库）

cd ../frontend
npm run build                  # 产出 dist/
npm run typecheck
```

---

## 配置

全部配置通过环境变量注入，`backend/src/main/resources/application.yml` 提供默认值。

| 变量 | 说明 |
| --- | --- |
| `DATABASE_URL` / `DB_USER` / `DB_PASSWORD` | PostgreSQL 连接 |
| `REDIS_HOST` / `REDIS_PORT` | Redis |
| `JWT_SECRET` | **生产必须替换**，≥32 字节 |
| `LLM_BASE_URL` / `LLM_API_KEY` / `LLM_CHAT_MODEL` / `LLM_EMBED_MODEL` | OpenAI 兼容模型网关（**密钥仅存在于服务端**） |
| `AI_CHAT_PROVIDER` / `AI_EMBED_PROVIDER` / `AI_VECTORSTORE_TYPE` | 设为 `none` 可关闭对应自动装配（离线运行） |
| `AI_MOCK_ENABLED` | `true` 强制走内置 Mock 引擎 |
| `OBJECT_STORAGE_ENDPOINT` / `ACCESS_KEY` / `SECRET_KEY` / `BUCKET` | MinIO；为空则导出写本地 `EXPORT_LOCAL_DIR` |
| `FREE_MONTHLY_UNITS` | free 套餐每月额度 |
| `ADMIN_EMAILS` | 管理员邮箱白名单（逗号分隔），为空时管理接口 403 |

### 模型供应商切换

业务层只依赖 Spring AI 抽象，切供应商只需改配置，无需改业务代码：

```yaml
spring:
  ai:
    model:
      chat: openai          # 或 none（离线）
      embedding: openai
    openai:
      base-url: ${LLM_BASE_URL}     # 任意 OpenAI 兼容网关
      api-key:  ${LLM_API_KEY}
      chat:
        options:
          model: ${LLM_CHAT_MODEL}
```

需要接国产云模型 / 本地 Ollama 时，引入对应 `spring-ai-starter-model-*` 并把 `spring.ai.model.chat` 改成对应值即可。

---

## API 概览

基础路径 `/api/v1`；成功响应 `{"data":...,"request_id":...}`，错误响应 `{"code":...,"message":...,"request_id":...}`。

| 方法与路径 | 用途 |
| --- | --- |
| `POST /auth/register` · `POST /auth/login` · `GET /auth/me` | 注册 / 登录 / 当前用户 |
| `GET\|POST /projects` · `GET\|PATCH\|DELETE /projects/{id}` | 作品管理 |
| `POST /projects/{id}/copy` · `POST /projects/{id}/archive` | 复制 / 归档 |
| `GET\|PATCH /projects/{id}/architecture` · `POST .../generate` | 雪花五步架构 |
| `GET\|POST /projects/{id}/outline` · `POST .../generate` | 章节大纲 |
| `GET\|POST /projects/{id}/entities` · `.../facts` | 设定库 / 剧情事实 |
| `GET /projects/{id}/graph` · `POST .../graph` · `.../snapshots` · `.../audit` | 灵感罗盘 |
| `POST /generations` · `GET\|DELETE /generations/{id}` · `GET .../stream` | 生成任务与 SSE |
| `GET\|PATCH /chapters/{id}` · `POST /chapters/{id}/review` | 章节与审校 |
| `GET\|POST /chapters/{id}/revisions` · `POST .../{no}/rollback` | 版本与回滚 |
| `POST /exports` · `GET /public/exports/{fileName}` | 导出与下载 |
| `GET /usage` · `GET /usage/jobs` | 用量与任务 |
| `GET /admin/audit-events` | 审计事件（管理员） |

错误码：`VALIDATION_ERROR(400)` `UNAUTHORIZED(401)` `FORBIDDEN(403)` `NOT_FOUND(404)` `VERSION_CONFLICT(409)` `QUOTA_EXCEEDED(402)` `RATE_LIMITED(429)` `MODEL_ERROR(502)` `MODEL_TIMEOUT(504)`。

---

## 提示词管理

所有提示词以 `.st` 资源文件放在 `backend/src/main/resources/prompts/`，启动时由 `PromptRepository` 载入，模板 ID 即相对路径（如 `architecture/core_seed`），并用内容哈希作为版本号记录到生成任务，支持后续 A/B 与回滚。

```
prompts/
├── architecture/   core_seed · character_dynamics · character_state
│                   world_building · plot_architecture
├── chapter/        blueprint · blueprint_chunked · first_draft · next_draft
│                   enrich · compress · rewrite · continue
├── compass/        extract_graph · extract_chapter_delta
│                   extract_chapter_graph · audit_graph
└── utility/        summary · update_character_state · review
```

---

## 数据模型

`users` · `novel_projects` · `project_architecture` · `outline_nodes` · `chapters` · `chapter_revisions` · `story_entities` · `plot_facts` · `graph_snapshots` · `graph_audits` · `generation_tasks` · `usage_ledger` · `audit_events` · `vector_store`

结构由 Flyway 管理（`spring.jpa.hibernate.ddl-auto=validate`），详见 `backend/src/main/resources/db/migration/`。

---

## 安全说明

- 所有作品相关接口在服务端校验所有权，越权返回 403/404。
- 模型 API Key 只存在于服务端环境变量，**不进入前端、响应体、日志或代码仓库**。
- 生成日志只保留排查字段；错误响应不回传堆栈。
- 审校/审计只输出建议与证据，不自动改稿、不自动锁定设定。
- 生产部署请替换全部默认密钥、启用 HTTPS，并配置数据库定期备份。

---

## 文档

| 文档 | 说明 |
| --- | --- |
| [功能说明](功能说明.md) | 完整功能、页面、AI 工作流、接口、数据模型、已知限制 |
| [环境搭建](环境搭建.md) | 从零开始的详细搭建步骤与排错手册 |
| [微信公众号宣传文](微信公众号宣传文.md) | 面向公众号渠道的产品宣传文案 |
| [测试报告](测试报告.md) | 真实环境端到端测试过程与结果（Docker + 真实链路 + 浏览器 UI 实测） |
| [问题与修复计划](问题与修复计划.md) | 测试中发现的问题、根因、解决方案与修复进度 |
| [README](README.md) | 本文件 |

---

## 已知限制

- 生成任务的可选参数暂存内存（`GenerationOptionsStore`），重启后未执行任务参数丢失；
- 管理后台后端仅提供审计事件接口，用户/成本/反馈为前端占位视图；
- 成本单价为代码内常量估算值，与真实账单存在偏差；
- 限流与预算告警、AI 固定评测集、真实作者封闭试点尚未落地；
- 前端为单 chunk 打包（约 1MB，含 ECharts / Naive UI），可按需引入优化；
- **离线 Mock 模式**：未配置模型 Key 时走内置 Mock 引擎（与真实链路同一套解析路径），输出为演示内容；接真实模型只需改配置；
- **SSE 收尾**：前端在收到终态事件后主动取消流式连接，浏览器控制台会记录 1 条 `net::ERR_ABORTED`（取消请求的正常网络记录，无功能影响）。

---

## 路线图

- [ ] 生成任务 `options` 落库（JSONB）
- [ ] 限流与预算告警（Redis 令牌桶 + 阈值告警）
- [ ] AI 固定冲突测试集与评测报告
- [ ] 管理后台补齐用户 / 成本 / 反馈
- [ ] 前端按需引入与分包
- [ ] 多模型分级路由（低成本模型处理摘要/审校）

---

## 许可与免责声明

本项目为工程落地参考实现。生成内容为辅助创作建议，**需作者人工审核**；不承诺内容绝对原创或不侵权、必然可出版。使用第三方模型服务时，请遵守对应供应商的服务条款与所在地法律法规。

---

## 贡献

欢迎提交 Issue 与 PR。提交前请确保：

```bash
cd backend && mvn -B test
cd ../frontend && npm run typecheck && npm run build
```
---

## 🏢 关于我们

<p align="center">
  <a href="http://www.net188.net">
    <img src="http://www.net188.net/images/logo1.png" alt="Net188 Logo" width="200" />
  </a>
</p>

<p align="center">
  <strong>Net188 · 互联网技术服务</strong>
</p>

<p align="center">
  专注于跨平台应用开发、AI Agent 集成与大模型应用落地。<br/>
  提供从产品设计、开发实施到部署运维的全栈技术解决方案。
</p>

<p align="center">
  🌐 <a href="http://www.net188.net"><strong>www.net188.net</strong></a>
</p>

---
