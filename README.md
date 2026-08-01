# 大学物理实验智能体 — RAG 知识库构建、导入与混合检索模块

## 一、项目简介
本模块为大学物理实验教学智能体提供**检索增强（RAG）能力**的数据底座与检索内核。
- **数据预处理**：基于 Excel 实验文档进行父子两层结构化切分，产出细粒度切片与完整父块的映射文件。
- **数据库直连导入**：采用**数据库直连脚本**，将父子 CSV 文件直接写入 PostgreSQL 数据库，保留 **Small-to-Big（父子分层检索）** 结构，让细碎切片真正参与后端检索。
- **后端双路混合检索**：依赖于后端 `app/services/rag_service_v2.py`，真实运用 **PostgreSQL ILIKE 关键词召回 + 7 信号重排序 + 父子上下文聚合**，返回标准化的 `chunk_id` 与完整父块背景，用于后检（PP-02）红线校验和智能体最终回答。

---

## 二、文件结构总览
```text
physics-lab-agent/                         # 项目根目录
│
├── external_data/                         # ⭐ 独立于后端的预处理目录
│   ├── build_chunk.py                     # 读取 Excel → 生成切分映射文件
│   ├── 大学物理实验.xlsx                   # 原始实验数据源（人工维护）
│   │
│   ├── child_mapping.csv                  # 子切块映射表（由 build_chunk.py 自动生成，检索核心入口）
│   └── parent_cache.csv                   # 父文档缓存表（由 build_chunk.py 自动生成，用于上下文回溯）
│
└── backend/                               # ⭐ 后端服务目录（容器内代码挂载点）
    ├── app/                               # FastAPI 核心源码（略）
    └── import_knowledge_via_db.py         # ⭐ 数据库直连导入脚本
```
## 三、模块分工详细说明
1. build_chunk.py（知识库预处理模块）
- 功能：读取 大学物理实验.xlsx 原始知识库，按实验目的、原理、步骤等 7 个维度完成父文档划分，对长文本进行滑动窗口切块（重叠 100 字符，子块上限 200 字符）生成细粒度子块，并调用 BAAI/bge-small-zh-v1.5 生成本地占位向量（作为验证副产品）。
- 产出：导出 parent_cache.csv、child_mapping.csv 两份配套映射文件。
- 注意：本脚本仅在本地离线执行，负责数据的结构化清理与准备。

2. import_knowledge_via_db.py（数据库直连导入模块）
- 位置：放置在 backend/ 目录下（与后端的 seed_poc_data.py 同级）。
- 功能：读取 parent_cache.csv 与 child_mapping.csv，通过 SQLAlchemy 直连 PostgreSQL 数据库，将数据真实注入。
- 数据落库：将 parent_cache.csv 的父块写入 knowledge_docs 表，作为全文背景。将 child_mapping.csv 的细碎子块写入 knowledge_chunks 表，形成独立的检索单元。核心优势：通过 document_id 将父子块在数据库层面进行强关联。这确保了的后端 RAG 引擎能够先检索到细碎子块，再回溯到完整父块。
- 运行环境：必须在容器内通过 docker exec 执行（因为依赖后端封装的数据库模型和连接池，不能在本地随意执行）。

3. 后端检索服务（核心接口）
本章节对应于后端中的 app/services/rag_service_v2.py 与 graph_agent_v2.py 的 _retrieve_evidence 方法。
检索请求由用户提问触发，通过以下严谨链路完成
- 实验隔离过滤：系统会从请求中提取 experiment_id（前端传的负数或数据库正数 ID 均可），在检索时自动转换为 experiment_code，并作为数据库 ILIKE 检索的硬过滤条件。保证了不同实验的知识库严格隔离。
- 双路混合检索策略：主路（真实运行时）：调用 rag_service_v2.search_chunks()。使用 jieba 中文分词 + PostgreSQL ILIKE 模糊匹配，在 knowledge_chunks 表的内容字段中召回最匹配的 Top-K 片段。辅路（预留/降级）：代码层集成了 milvus_service。若需要启用语义检索，可将向量化后的用户问题存入 Milvus 进行相似度搜索，作为关键词检索的补充（当前项目实测阶段默认降级走 PG 关键词）。
- 多信号重排序（7 信号 Rerank）：从数据库中召回的多个候选 chunk 会经过一个加权打分模型：Token 命中度（Jaccard 相似度与词频，占 35%）、占位哈希向量余弦相似度（20%）、章节类型相关性（按原理、步骤、安全等类型动态加权，15%）、版本一致性（10%）、文档权威性（按 review_status 判断，10%）、标题命中加分（10%）、内容长度惩罚

4.父子上下文扩展
- 由于import_knowledge_via_db.py脚本将 child_mapping.csv 真实落库，后端检索命中子块后，代码中的 _expand_with_parent 逻辑会被触发。
- 它会根据 chunk.document_id 立即从 knowledge_docs 表中拉取完整的父块原文，拼接到检索结果中。这让 LLM 既能感知细粒度的精准检索，又能获取完整的上下文背景。

5.结构化引用返回
- 最终检索结果以标准 evidence 字典返回，包含 items 列表（含 chunk_id、document_id、document_version、location、content 等）。这些 chunk_id 会在后续的 PP-02 后检规则 中被严格核验。

## 四、工作流与数据流转
```text
1. 数据生成阶段（用户本地操作）
  大学物理实验.xlsx
      │
      ▼ (执行 python build_chunk.py)
  生成 parent_cache.csv 与 child_mapping.csv
      │
      ▼ 
2. 文件拷贝与导入阶段（容器交互）
  [本地终端] docker cp 复制 CSV 与脚本文件到容器 /app/ 目录下
      │
      ▼ 
  [容器内部] docker exec -it physics-backend sh
      │
      ▼ (执行 python /app/import_knowledge_via_db.py)
  写入 PostgreSQL 业务库 (KnowledgeDoc & KnowledgeChunk)
      │
      ▼ (入库完成，父子关系持久化绑定)
3. 在线检索阶段（用户提问）
  ChatPanel.vue 提问
      │
      ▼
  graph_agent_v2._retrieve_evidence() 触发
      │
      ▼ 
  [Redis 缓存命中(基于 exp_id+query_hash, TTL 300s)] → 直接返回
      │
      ▼ 
  未命中 → [PostgreSQL ILIKE 关键词召回子块]
      │
      ▼ 
  合并候选池 → [7 信号重排序] → [根据 doc_id 回溯父块并聚合]
      │
      ▼ 
  返回 evidence 对象 (含 chunk_id 与父块全文)
      │
      ▼ 
4. 回答输出阶段
  LLM 生成答案 → 后检 PP-02 校验 chunk_id 引用 → 返回前端 citations
```
## 五、环境依赖
1. 预处理端（本地 Python 环境）
- 执行 build_chunk.py 前，请确保已安装以下 Python 包：pandas openpyxl sentence-transformers
2. 导入与检索端（后端 Docker 环境）
- 导入脚本 import_knowledge_via_db.py 依赖后端 backend/requirements.txt 中的包（sqlalchemy[asyncio]、asyncpg 等），因此必须在容器内执行，本地环境无需配置。

## 六、快速运行指南
### 1. 构建映射文件（生成数据源）
```
bash

# 进入 external_data 目录
cd external_data
python build_chunk.py
```
(运行后会在当前目录生成 parent_cache.csv 和 child_mapping.csv。)
### 2. 将脚本与文件复制进容器
确保后端容器已开启（docker compose up -d backend），在项目根目录执行：
```
bash

# 1. 把新建的脚本复制进容器
docker cp backend/import_knowledge_via_db.py physics-backend:/app/
# 2. 把两张 CSV 文件也复制进容器
docker cp external_data/parent_cache.csv physics-backend:/app/
docker cp external_data/child_mapping.csv physics-backend:/app/
```
### 3. 进入容器执行数据库直连导入
```
bash

# 进入后端的交互式终端
docker exec -it physics-backend sh
# 在容器内运行导入脚本
python /app/import_knowledge_via_db.py
```
(控制台会打印详细的日志：“✅ 创建父文档: xxx”、“🔹 创建子块: chunk_import_xxx”等，最终提示导入完成。)
### 4. 前端验证父子检索链路
导入完成后，打开浏览器（http://localhost:8080），登录账号，切换到已导入数据的实验。
在聊天框提问：“这个实验的【原理】是什么？”

重点观察 AI 的思考过程和最终回答中的引用字段。如果出现 chunk_import_xxx 格式的 chunk_id，即证明：

1.子块已成功入库并被检索命中。

2.学长的 RAG 引擎已成功回溯到父块并将其作为上下文喂给了 AI。

3.后检 PP-02 规则已成功校验了引用。Small-to-Big 链路 100% 打通！

## 七、注意事项
实验名映射表：import_knowledge_via_db.py 脚本内的 EXPERIMENT_CODE_MAP 变量记录了“实验中文名”与后端数据库 experiment_code 的映射关系。如果后续在 CSV 中新增了 Excel 表里没有的实验名称，必须在此处手动补全对应的 code，否则脚本会直接跳过。
