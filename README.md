# 企业级智能客服工单系统

> 基于 Dify 开源 AI 应用平台二次开发，融合 RAG 知识库 + 工单流转的一体化智能客服系统

## 项目概述

本项目面向企业客户服务场景，基于开源 AI 应用平台 Dify 进行深度定制与私有化部署，构建「AI 前置应答 + 人工兜底处理」的服务闭环。系统通过向量检索 + 关键词混合检索实现企业产品手册、规章制度、FAQ 文档的智能问答，结合意图识别与多轮对话上下文管理，自动处理高频咨询；复杂问题自动触发工单创建、智能分派、状态追踪与超时提醒，形成完整的客服业务闭环。

### 解决的业务痛点

- **人工客服成本高** — AI 自动应答高频重复问题，降低人工坐席压力
- **咨询重复率高** — RAG 知识库精准匹配企业文档，避免重复人工解答
- **私有问题无法解答** — 本地部署私有模型 + 企业专属知识库，数据不出内网
- **工单流转低效** — AI 自动分类、分派、追踪工单状态，减少人工调度

## 技术架构

```
┌──────────────────────────────────────────────────────────────┐
│                        Nginx (反向代理)                        │
│                      HTTP / HTTPS / WebSocket                  │
└──────────┬───────────────────────────┬───────────────────────┘
           │                           │
    ┌──────▼──────┐            ┌───────▼───────┐
    │  Dify Web   │            │  Dify API     │
    │  (Next.js)  │            │  (Flask)      │
    └──────┬──────┘            └───────┬───────┘
           │                           │
    ┌──────▼───────────────────────────▼───────┐
    │            Dify Worker (Celery)           │
    │    + 自定义工单引擎 (Ticket Engine)         │
    └──────┬───────────────────────────┬───────┘
           │                           │
    ┌──────▼──────┐            ┌───────▼───────┐
    │  PostgreSQL │            │    Redis      │
    │  (业务数据)  │            │  (缓存/队列)   │
    └──────┬──────┘            └───────┬───────┘
           │                           │
    ┌──────▼───────────────────────────▼───────┐
    │            Weaviate (向量数据库)            │
    │       存储企业知识库文档 Embedding          │
    └───────────────────────────────────────────┘
           │
    ┌──────▼──────┐
    │   Ollama    │
    │  Qwen2.5:7b │
    │ (本地推理)   │
    └─────────────┘
```

## 技术栈

| 层级 | 技术 | 说明 |
|------|------|------|
| 前端 | Next.js (React) | Dify Web UI |
| 后端 | Flask (Python) | Dify API + 工单引擎 |
| 任务队列 | Celery + Redis | 异步任务、工单流转、超时检测 |
| 数据库 | PostgreSQL | 业务数据、工单、用户 |
| 向量数据库 | Weaviate | 知识库文档向量存储与检索 |
| 本地模型 | Ollama + Qwen2.5:7b | 意图分类、RAG 检索增强、多轮对话 |
| 云端模型 | Qwen3.5-Flash (DashScope) | 工单信息提取 |
| 部署 | Docker Compose | 容器化一键部署 |

## 核心功能

### 1. 智能问答（RAG 知识库）

- 基于 Weaviate 向量数据库 + 关键词混合检索
- 企业产品手册、规章制度、FAQ 文档的语义理解与精准匹配
- 多轮对话上下文管理，保持对话连贯性
- 向量检索与关键词检索的分值加权排序

### 2. 意图识别与分类

- 本地 Ollama Qwen2.5:7b 模型进行在线意图分类
- 支持多分类场景：售后咨询、产品咨询、技术故障、投诉建议等
- 置信度阈值判定，低分自动转人工

### 3. 工单流转系统

- AI 自动提取工单关键信息（问题描述、客户信息、优先级等）
- 智能分派至对应处理部门（售后/技术/产品）
- 工单状态追踪：待处理 → 处理中 → 已解决 → 已完成
- 超时未处理自动提醒（Celery 异步扫描）

### 4. 私有化部署

- 全内网部署，零外部 API 依赖（可选 DashScope 云端模型增强）
- Ollama 本地模型推理，数据不出企业内网
- 向量数据库本地存储，文档索引完全可管控

### 5. 标准化 API 接口

- 工单创建接口：`POST /api/ticket/create`（带重试机制）
- 意图分类接口
- 知识库检索接口
- 兼容 OpenAI API 格式，方便集成

## 快速部署

### 前提条件

- Docker & Docker Compose
- 服务器配置不低于 2 核 4GB 内存（推荐 4 核 16GB）
- 内网环境需提前部署 Ollama 并拉取模型

### 部署步骤

```bash
# 1. 进入 Docker 部署目录
cd docker

# 2. 配置环境变量
cp .env.example .env
# 编辑 .env 文件，配置数据库、Redis、向量数据库等参数

# 3. 启动所有服务
docker compose up -d

# 4. 初始化上线（首次需要进入容器初始化 Ollama 模型）
docker compose exec api ollama pull qwen2.5:7b
```

> 详细部署说明请参考 [Docker 部署指南](./docker/README.md)

## 运维命令

```bash
# 查看服务状态
docker compose ps

# 查看日志
docker compose logs -f api
docker compose logs -f worker

# 重启服务
docker compose restart worker

# 备份数据库
docker compose exec db_postgres pg_dump -U postgres dify > dify_backup_$(date +%Y%m%d).sql

# 备份 Weaviate 向量数据
docker compose cp weaviate:/var/lib/weaviate ./backup/weaviate_data/
```

### 服务扩缩容

```bash
# 调整 Worker 数量（编辑 .env 中的 CELERY_WORKER_AMOUNT）
CELERY_WORKER_AMOUNT=8

# 重启 Worker 生效
docker compose restart worker
```

## 项目亮点

1. **企业级业务闭环** — 不同于普通 AI 问答 Demo，完整覆盖 RAG 知识库 + 意图识别 + 工单流转 + 超时提醒，高度贴合工业级客服场景落地
2. **全私有化改造** — Ollama 本地模型 + 内网向量数据库 + 私有存储，零外部 API 依赖，完全满足企业数据安全与合规要求
3. **工程化落地经验** — AI 问答自动化、业务系统对接（官网/公众号/OA）、数据监控运维一体化，具备完整的大模型应用工程化能力
4. **可扩展架构** — 基于 Dify 工作流引擎与 Celery 异步任务，支持水平扩缩容，可支撑企业常态化客户服务与内部咨询场景

## 参考文档

- [Dify 官方文档](https://docs.dify.ai/)
- [Ollama 模型库](https://ollama.com/library)
- [Weaviate 文档](https://weaviate.io/developers/weaviate)
- [Celery 文档](https://docs.celeryq.dev/)

## License

本项目基于 Dify 开源版本（Apache 2.0）进行二次开发，继承原有开源协议。
