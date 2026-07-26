# 大学物理实验智能体 — RAG 知识库检索模块
## 项目简介
### 本模块为**大学物理实验教学智能体**提供检索增强（RAG）能力，采用 **Small-to-Big（父子分层检索）** 方案。
### 对实验文档进行细粒度子块切片检索，最终返回完整父块上下文，解决单纯细切片语义断裂问题；内置实验预过滤逻辑，减少跨实验无关内容召回。
### 向量存储使用 FAISS 本地向量库，嵌入模型选用 BAAI/bge-small-zh-v1.5，全程支持离线运行。当前全局召回率：90.57%（50 条测试集全量验证）。
## 文件结构说明
### ├── small\_big\_retrieve.py   #分层RAG：文本切片、构建向量库
### ├── rag\_api.py              # RAG检索HTTP接口（供智能体主项目调用）
### ├── test\_api.py             # 接口本地调试测试脚本
### ├── recall\_retrieve.py      # 召回率批量评测脚本 
### ├── 大学物理实验.xlsx        # 原始知识库数据源
### ├── test\_dataset.xlsx       # 评测测试集（带标注相关切片ID）
### ├── child\_index.faiss       # FAISS向量索引文件（代码运行生成）
### ├── child\_mapping.csv       # 子切片id-文本映射表
### ├── parent\_cache.csv        # 父块缓存映射表
### ├── rag\_recall\_stat\_result.xlsx # 召回评测输出报告（自动生成）
## 模块分工说明
### 1.small\_big\_retrieve.py
- 读取 Excel 知识库
- 父文档划分 + 子文本切片
- 向量化，构建 / 加载 FAISS 向量库
- 实现分层语义检索、实验预过滤
