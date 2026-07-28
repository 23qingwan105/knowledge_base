# 大学物理实验智能体 — RAG 知识库检索模块
## 项目简介
### 本模块为大学物理实验教学智能体提供检索增强（RAG）能力，采用 Small-to-Big（父子分层检索） 优化方案。
- 对实验文档进行细粒度子块切片检索，最终拼接完整父块上下文，解决单纯细切短句语义断裂问题；
- 内置实验预过滤逻辑，大幅减少跨实验无关内容召回；
- 向量底座使用 FAISS 本地向量库，嵌入模型选用 BAAI/bge-small-zh-v1.5，全程支持离线运行；
- 全量 50 条测试集召回验证，整体召回准确率：90.57%。
## 一、文件结构总览
### 1.build_chunk.py
分层 RAG 建库脚本，只负责知识库预处理，不包含检索逻辑。功能：读取 Excel 知识库、拆分父文档与子文本切块、加载 bge 向量化文本、构建并持久化 FAISS 向量索引、自动生成 child_mapping.csv、parent_cache.csv 两份映射缓存文件。
### 2.retrieve_ref.py
分层检索核心文件，封装 SmallBigRetriever 检索类，完整实现 Small-to-Big 父子检索逻辑。支持本地直接调试运行，可输入问题完成子块向量召回、父文档聚合、完整上下文拼接，附带本地检索测试入口。
### 3.recall_eval.py
召回率批量评测脚本，读取标注测试集自动批量执行检索，统计召回命中情况，自动输出召回指标，落地生成 rag_recall_stat_result.xlsx 评测报表。
### 4.大学物理实验.xlsx
原始知识库数据源，所有实验完整资料的源头文件，人工维护更新。
### 5.test_dataset.xlsx
人工标注评测测试集，每条查询语句绑定正确对应的子切片 ID，用来量化校验整体召回效果。
### 6.rag_recall_stat_result.xlsx
执行 recall_eval.py 后自动产出，包含每条测试用例命中明细、整体召回汇总数据，用于效果复盘迭代。
## 二、程序自动生成的索引 & 映射文件
### 1.child_index.faiss
FAISS 向量索引文件，存储全部子切块文本向量，由 build_chunk.py 运行生成，用于向量相似度检索。
### 2.child_mapping.csv
子切块映射表，记录每一条子块 ID、对应文本、归属父文档 ID，代码自动生成，不可手动修改。
### 3.parent_cache.csv
父文档缓存表，保存所有父文档完整原文、实验名、板块分类；检索命中子块后依靠本表快速读取完整父上下文，自动生成。
## 三、模块分工详细说明
### 1. build_chunk.py（知识库构建模块，无检索能力）
- 读取大学物理实验.xlsx原始知识库，完成父文档划分、子文本滑动窗口切块；
- 调用 BAAI/bge-small-zh-v1.5 完成所有切块文本向量化；
- 新建 / 覆盖本地 FAISS 向量库 child_index.faiss；
- 自动导出 child_mapping.csv、parent_cache.csv 两份配套映射文件；
- 仅做离线建库，不包含任何查询检索逻辑。
### 2. retrieve_ref.py（检索核心模块）
- 定义 SmallBigRetriever 检索类，加载 FAISS 索引、两份 csv 映射文件；
- 完整实现 Small-to-Big 分层检索：先召回相似度最高的细粒度子块，对子块归属父 ID 去重，批量读取父文档完整内容；
- 返回结构化结果：父 ID、实验名称、板块、完整原文，上层业务可直接取用；
- 自带 main 测试入口，支持本地输入问题一键调试检索效果。
### 3. recall_eval.py（召回量化评测模块）
- 读取标注好的 test_dataset.xlsx 批量执行检索；
- 自动比对检索结果和标注正确切片，计算整体召回率；
- 全量测试结果写入 rag_recall_stat_result.xlsx，方便批量查看每条用例好坏，辅助优化切块、过滤规则。
### 4. 各类资源文件说明
- 手动维护文件：大学物理实验.xlsx（唯一需要人工更新）、test_dataset.xlsx（评测标注，固定不变）；
- 运行 build_chunk.py 自动生成：child_index.faiss、child_mapping.csv、parent_cache.csv，无需手动编辑；
- 运行 recall_eval.py 自动生成：rag_recall_stat_result.xlsx 评测报表。
## 四、环境依赖
- faiss-cpu
- sentence-transformers==2.7.0
- pandas
- openpyxl
- numpy
## 五、快速运行指南
### 1.初始化构建向量库与映射文件
python build_chunk.py
- 读取原始实验 Excel，一次性生成 FAISS 索引、两份 CSV 映射文件。后续知识库改动都需要重新执行本脚本刷新索引。
### 2.本地调试检索功能
python retrieve_ref.py
- 脚本内置测试 query，运行即可查看检索返回的完整父上下文；如需测试自己的问题，修改代码内 test_query 内容即可。
### 3.批量自动化评测召回指标
python recall_eval.py
- 跑完控制台打印召回率，同时生成 rag_recall_stat_result.xlsx 完整评测表格。
## 六、更新备注
