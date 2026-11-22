# EchoMind

**多 Agent 智能客服后端**

EchoMind 面向通用咨询、技术支持和账单售后场景，将意图识别、角色协作、知识检索、工具调用与会话记忆整合为统一的服务链路，提供 HTTP API、标准 MCP 接口和运行监控。

本仓库包含 Python 后端。配套 Web 前端与 Java 后端位于独立项目中；本服务可通过 HTTP 客户端或 Swagger 独立使用。

<a id="overview"></a>

## 核心能力

| 模块 | 能力 |
|---|---|
| 意图识别 | 融合 LLM、Embedding 与规则，识别业务领域、关键实体及紧急度 |
| Agent 协作 | General、Technical、Billing、Escalation 角色分工，支持主辅角色并行处理与回复合成 |
| 知识检索 | Agent 按需调用 RAG 工具，支持查询改写、多路召回和可选重排 |
| 会话记忆 | Redis 保存近期消息和摘要，ChromaDB 保存情景记忆与用户画像 |
| 业务规则 | 从 Skills 文件加载处理规范，按角色与关键词匹配，支持运行时重载 |
| 工具接入 | 标准 MCP、工具白名单、参数校验、缓存、超时与熔断 |
| 可靠性 | 会话并发控制、重复请求回执、整轮原子提交和故障回退 |
| 质量与观测 | 异步评测、LLM-as-Judge、回归基线、调用轨迹和 Prometheus 指标 |

账单工具当前支持核验字段检查与金额差值计算，人工升级生成交接摘要。订单查询、退款执行和工单创建需要接入对应业务系统。

## 系统设计

```text
HTTP 请求 → 鉴权与回执检查 → 读取会话上下文 → 意图识别与实体提取
                                              ↓
                                单角色处理 / 主辅 Agent 协作
                                              ↓
                                  Skills · RAG · MCP 工具
                                              ↓
                                回复合成 → 原子保存 → 返回响应
                                              ↓
                                    后台记忆维护与运行监控
```

技术栈：Python · FastAPI · Anthropic 兼容模型接口 · Redis · ChromaDB · MCP · Docker Compose · Prometheus。

架构图与处理细节见 [系统架构](wiki/架构图.md)、[业务流程](wiki/业务流程说明.md) 和 [技术设计](wiki/技术亮点.md)。

<a id="quickstart"></a>

## 运行

环境要求：Docker、Docker Compose v2，以及支持 Anthropic Messages 协议和工具调用的模型服务。

复制配置文件并填写模型服务信息：

```bash
cp .env.example .env
```

PowerShell 使用 `Copy-Item .env.example .env`。已有 `.env` 时直接更新所需配置。

| 配置 | 说明 |
|---|---|
| `ANTHROPIC_API_KEY` | 模型服务密钥 |
| `ANTHROPIC_MODEL` | 账号可调用的模型 ID |
| `ANTHROPIC_BASE_URL` | Anthropic 兼容地址；使用官方服务时可留空 |
| `ECHOMIND_ENV` | 本地为 `development`；对外部署使用 `production` |
| `ECHOMIND_API_TOKENS` | 生产环境的用户与管理员访问凭据 |
| `SILICONFLOW_API_KEY` | 可选的在线意图向量服务密钥 |

```bash
docker compose up -d --build
docker compose ps
docker compose logs -f echomind
```

默认启动 API、Redis、ChromaDB、Prometheus 和 Nginx。

| 入口 | 地址 |
|---|---|
| API / Swagger | `http://localhost:8000/docs` |
| 健康检查 | `http://localhost:8000/health` |
| Nginx 入口 | `http://localhost/docs` |
| Prometheus | `http://localhost:9090` |

`/health` 检查应用初始化状态；模型连通性通过实际对话确认。Web 配置、存储管理和部署参数见 [部署与使用](wiki/完整使用指南.md)。

<a id="usage"></a>
<a id="mcp"></a>
<a id="skills"></a>
<a id="evaluation"></a>

## 服务接口

| 接口 | 用途 |
|---|---|
| `POST /chat` | 发送消息并返回回复、路由信息和请求标识 |
| `POST /search` | 查询知识库 |
| `GET /skills` · `POST /skills/reload` | 查看和重新加载业务规则 |
| `GET /integrations/mcp` | 查看外部 MCP 接入状态 |
| `POST /eval/run` · `GET /eval/jobs/{job_id}` | 提交评测并读取任务结果 |
| `GET /monitor` · `GET /metrics` | 查看运行统计与 Prometheus 指标 |

请求结构以 Swagger 为准。启用访问控制后使用 `Authorization: Bearer <token>`，管理接口需要管理员身份。

<a id="development"></a>

## 开发与验证

```bash
python -m pip install -r requirements-dev.txt
python -m pytest -q
```

集成检查与实际模型调用所需环境见 [部署与使用](wiki/完整使用指南.md)。

```text
api/          HTTP API、初始化与 CLI
agents/       Agent 编排、角色工具与回复合成
core/         意图识别、会话状态、鉴权与性能采集
memory/       会话记忆与用户画像
tooling/      RAG、工具管理与 MCP
evaluation/   评测、评分校准与任务队列
monitor/      监控指标与路由反馈
skills/       业务处理规范
config/       服务与监控配置
tests/        自动化测试与集成检查
wiki/         产品、架构与运维文档
```

<a id="configuration"></a>
<a id="intent-embedding-setup"></a>
<a id="deployment"></a>
<a id="faq"></a>
<a id="documentation"></a>

## 文档

- [部署与使用](wiki/完整使用指南.md)：配置、在线 Embedding、接口、监控和故障排查。
- [定位与技术亮点](wiki/EchoMind定位与技术亮点.md)：产品能力与设计取舍。
- [Skills 业务规则](skills/README.md)：规则格式与加载方式。
- [场景扩展](wiki/EchoMind%20场景扩展指南.md)：业务场景接入方案。
- [文档中心](wiki/文档中心.md)：完整文档索引。
