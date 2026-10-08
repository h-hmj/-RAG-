# 智扫通机器人智能客服

基于 **LangChain** 和 **RAG (检索增强生成)** 技术构建的智能客服 Agent 项目。本项目实现了一个能够结合本地知识库、调用外部工具并根据场景动态切换提示词的智能助手，通过 Streamlit 提供 Web 交互界面。

## 项目特性

- **RAG 知识库问答**：集成 ChromaDB 向量数据库，支持 PDF/TXT 格式知识文件的自动加载、分片与检索，解决大模型知识盲区。
- **ReAct Agent 架构**：基于 LangChain `create_agent` 构建，具备推理与行动能力，可根据用户意图自动调用工具。
- **丰富的工具集**：
  - `rag_summarize`: 知识库检索与总结。
  - `get_weather`: 模拟天气查询。
  - `get_user_location` / `get_user_id`: 获取用户上下文信息。
  - `fetch_external_data`: 查询用户历史使用数据（CSV）。
  - `fill_context_for_report`: 触发报告生成场景的上下文注入。
- **动态提示词中间件**：通过 Middleware 机制监控工具调用，当检测到“生成报告”意图时，自动切换 System Prompt 为报告专用模板。
- **流式输出**：支持打字机效果的流式回复，提升用户体验。
- **增量知识库加载**：基于文件 MD5 校验，避免重复向量化，支持知识库增量更新。

## 项目结构

```text
.
├── app.py                  # Streamlit 应用入口
├── agent/                  # Agent 核心逻辑
│   ├── react_agent.py      # ReAct Agent 封装
│   └── tools/
│       ├── agent_tools.py  # 自定义工具集
│       └── middleware.py   # 中间件（日志、监控、动态提示词）
├── rag/                    # RAG 模块
│   ├── rag_service.py      # 检索与总结服务
│   └── vector_store.py     # 向量库管理与文档加载
├── model/                  # 模型工厂
│   └── factory.py          # 初始化 LLM 和 Embedding 模型
├── utils/                  # 工具类
│   ├── config_handler.py   # YAML 配置读取
│   ├── prompt_loader.py    # 提示词加载
│   ├── path_tool.py        # 路径处理
│   ├── file_handler.py     # 文件解析 (PDF/TXT)
│   └── logger_handler.py   # 日志配置
├── config/                 # 配置文件目录
│   ├── rag.yml             # 模型配置
│   ├── chroma.yml          # 向量库配置
│   ├── prompts.yml         # 提示词路径配置
│   └── agent.yml           # Agent 数据源配置
├── prompts/                # 提示词文件存储
├── data/                   # 知识库源文件 (PDF/TXT)
├── chroma_db/              # 向量数据库持久化目录
└── logs/                   # 运行日志
```

## 快速开始

### 1. 环境准备

- Python 3.10+
- 阿里云 DashScope API Key (通义千问)

### 2. 安装依赖

```bash
pip install streamlit langchain langchain-openai langchain-chroma chromadb pyyaml pypdf
```

### 3. 配置环境变量

设置 API Key 以访问大模型服务：

```bash
# Windows PowerShell
$env:OPENAI_API_KEY="your-dashscope-api-key"

# Linux/Mac
export OPENAI_API_KEY="your-dashscope-api-key"
```
*(注：项目使用 OpenAI 兼容模式访问 DashScope，需设置 OPENAI_API_KEY)*

### 4. 初始化知识库

将知识文档放入 `data/` 目录，然后运行以下命令构建向量索引：

```bash
python -m rag.vector_store
```

### 5. 启动应用

```bash
streamlit run app.py
```

## 配置说明

项目配置集中在 `config/` 目录下的 YAML 文件中：

- **rag.yml**: 配置 Chat 模型 (`chat_model_name`) 和 Embedding 模型 (`embedding_model_name`)。
- **chroma.yml**: 配置向量库集合名、持久化路径、分片参数 (`chunk_size`) 及知识库数据源路径。
- **prompts.yml**: 指定主提示词、RAG 提示词和报告提示词的文件路径。
- **agent.yml**: 配置外部数据文件 (CSV) 的路径。

## 核心流程

1. **用户输入**：通过 Streamlit 界面输入问题。
2. **Agent 思考**：`ReactAgent` 接收输入，结合 System Prompt 进行推理。
3. **工具调用**：
   - 若需查询知识，调用 `rag_summarize` -> 触发 `VectorStore` 检索 -> LLM 总结。
   - 若需生成报告，调用 `fill_context_for_report` -> 触发 **中间件** 修改 Runtime Context。
4. **动态切换**：中间件 `report_prompt_switch` 检测到 Context 标记，将后续 LLM 调用的 Prompt 切换为报告专用模板。
5. **流式响应**：结果以流式形式返回给前端展示。
