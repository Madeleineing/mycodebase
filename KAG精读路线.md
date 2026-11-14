# KAG 精读路线：不复现，但打好科研基础

**适用目标**：理解方法本身 + 建立 RAG/知识图谱方向的可迁移判断力，为后续选题做准备。
**总投入**：约 5–8 天（每天 2–3 小时）。
**核心原则**：读 prompt 和 schema，而不是读 pipeline 代码。方法写在前者里，后者只是编排。

---

## 零、先建立坐标系（半天）

| 代际 | 做法 | 遗留问题 |
|---|---|---|
| 朴素 RAG | 切块 + 向量相似度检索 | 语义漂移，多跳问答失效 |
| GraphRAG | OpenIE 自由抽取建图 + 社区摘要 | 噪声多，实体严重冗余 |
| **KAG** | schema 约束建图 + 逻辑形式引导推理 | schema 依赖专家，构建成本高 |

**必读的对照论文**（不要求全读，读摘要 + 主图 + 实验表即可）：
- RAG（Lewis 2020）—— 奠基
- GraphRAG（Edge 2024,微软）—— 直接对手
- HippoRAG（2024）—— 另一条"用图做多跳"的路线
- KAG（Liang 2024）—— 本论文

读完这一圈，你要能一句话说清：**KAG 和 GraphRAG 的根本分歧在"要不要 schema"**。
GraphRAG 认为 schema 太重，用 OpenIE 自由抽取；KAG 认为自由抽取噪声太大，宁可要 schema 约束。
这个分歧本身就是可以写文章的争议点。

---

## 一、三层阅读法

### 第 1 层：论文层（1 天）

不要逐字精读。只读这四处：

1. **摘要 + Introduction** —— 抓"它声称解决了什么"
2. **Figure 2（整体架构图）** —— 抓 kg-builder / kg-solver 的分工
3. **知识表示那节（LLMFriSPG）** —— 抓 DIKW 分层：数据 → 信息 → 知识 → 智慧
4. **实验主表** —— 抓它在 HotpotQA / 2Wiki / MuSiQue 上比谁高多少

**合上论文，你必须能回答这 3 个问题**：
- 它的四个核心 contribution 分别对应哪个技术点？
- 它比 GraphRAG 强，靠的是哪一步？
- 它的实验设置里，LLM 用的是什么？（答：qwen2.5-72b，这点很重要——说明效果强依赖大模型能力）

> 提醒：读实验表时留意指标是 EM/F1。这类指标对"答案表述"极其敏感，
> 高分不完全等于推理能力强。这是你建立批判性阅读习惯的第一课。

---

### 第 2 层：语义层（1–2 天）★ 最关键，也最省时间

这一层**完全不需要跑环境**，直接读文件。KAG 的方法论几乎全在这里：

#### 2.1 读 schema —— 理解"知识表示"

```
kag/open_benchmark/hotpotqa/schema/HotpotQA.schema
kag/examples/medicine/schema/Medicine.schema      ← 对比看，感受领域 schema 的差异
```

读的时候盯三件事：
- 实体类型怎么划分？（HotpotQA 是 16 个大类：Person / Works / GeographicLocation...）
- 每个属性后面挂着什么索引？（`index: TextAndVector` —— 这就是"互索引"的落地）
- `semanticType(语义类型)` 和 `synonyms(同义词)` 这两个属性是干嘛的？
  → 这就是论文说的**知识对齐**：把抽取出的实体挂到语义概念树上，消歧 + 合并同义项。

**关键领悟**：schema 是 KAG 的灵魂，也是它的软肋。领域 schema 得靠专家手写——
`Medicine.schema` 你得有医学知识才写得出来。这就是"构建成本高"的根源。

#### 2.2 读 prompt —— 理解"方法本体"

按这个顺序读，每个 5–10 分钟：

| 文件 | 对应论文哪个部分 |
|---|---|
| `kag/solver/prompt/static_planning_prompt.py` | 逻辑形式规划：把问题拆成算子 DAG |
| `kag/solver/prompt/logic_form_plan.py` | 逻辑形式的符号化表达 |
| `kag/solver/prompt/spo_retrieval.py` | 图上 SPO 检索 |
| `kag/solver/prompt/resp_judge.py` / `resp_verifier.py` | 答案自校验 |
| `kag/builder/prompt/default/ner.py` | 实体抽取 |
| `kag/builder/prompt/default/triple.py` | 三元组抽取 |
| `kag/builder/prompt/default/std.py` | 属性标准化（对齐） |
| `kag/builder/prompt/outline_prompt.py` | 大纲索引（0.8 新增） |

**读 `static_planning_prompt.py` 时重点看它的 example**：

```
query: "Which movies have Jacky Cheung and Andy Lau starred in together?"
output:
  "0": { executor: "Retriever", arguments: { query: "..." } }
  "1": { executor: "Retriever", arguments: { query: "..." } }
  "2": { executor: "Code", dependent_task_ids: ["0","1"],
         arguments: { query: "写 Python 找两个列表的交集：{{0.output}} {{1.output}}" } }
```

看懂这一段，你就懂了 KAG 的整个推理范式：
**LLM 不直接回答问题，而是产出一个算子 DAG；系统按拓扑序执行，后一步引用前一步的输出。**
这个思想不只属于 KAG，ReAct、Toolformer、各类 Agent 都是同一个内核。这是你能带走的最有价值的东西。

---

### 第 3 层：结构层（2–3 天）

只读**一条**完整链路，不要横向铺开。目标是理解"系统怎么组织"，不是"所有组件怎么实现"。

1. `kag/interface/` —— 所有抽象基类。搞清角色划分：Reader / Splitter / Extractor / Vectorizer / PostProcessor / Writer
2. `kag/open_benchmark/hotpotqa/kag_config.yaml` —— 一份完整配置，作为你的地图
3. `kag/builder/default_chain.py` —— 组件怎么用 `>>` 串成链
4. `kag/solver/pipeline/kag_static_pipeline.py` —— 推理主流程
5. `kag/solver/executor/retriever/` —— 检索算子（最核心的 executor）

**关键机制**：YAML 里写 `type: unstructured_builder_chain`，代码里对应
`@KAGBuilderChain.register("unstructured_builder_chain")`（`default_chain.py:70`）。
所有组件靠字符串查表装配。抓住这一点，9.8 万行代码就变成了一张可以按需查询的表。

**可以整个跳过**：`knext/`（3.7 万行，只是跟服务端通信的 SDK，跟方法论无关）。

---

## 二、不复现，但可以做的"轻量验证"

这几件事都不需要 OpenSPG 环境：

1. **单跑一个 prompt**：用任意 OpenAI-compatible 的 API（甚至本地 ollama），
   把 `static_planning_prompt.py` 里的中文模板喂进去，随便问个多跳问题，
   看大模型返回的 DAG 长什么样。花 20 分钟，但你会对"逻辑形式"有实感。

2. **手画一张数据流图**：从 `corpus.json` 的一篇文档出发，
   画一遍它经过 reader → splitter → extractor → vectorizer → writer 之后变成什么。
   画不出来的地方，就是你没真懂的地方。

3. **做一份"组件 ↔ 论文贡献"对照表**，填不上的格子就是你的疑问清单。

---

## 三、可迁移的收获：拆解任意一篇 RAG 论文的 5 个问题

这才是"打基础"的真正目标。以后你读任何一篇这个方向的论文，都用这 5 个问题套：

1. **知识从哪来？** —— 无 schema 自由抽取 / schema 约束 / 已有 KG / 结构化数据
2. **知识怎么存？** —— 纯向量 / 图 / 图 + 文本互索引 / 分层索引（大纲、摘要、原子问题…）
3. **检索的粒度是什么？** —— chunk / 实体 / 三元组 / 子图 / 社区摘要
4. **推理怎么组织？** —— 单轮检索 / 多轮迭代 / 逻辑形式 DAG / Agent 自主规划
5. **效果提升到底来自哪一步？** —— 换更强的 LLM？还是方法本身？
   （很多论文的提升其实来自基座模型变强，这是最常见的陷阱）

把这 5 个问题套到 KAG 上，你就能写出它的"方法卡片"。
以后读到 LightRAG、HippoRAG2、GraphRAG 的新版本，都往同一张表里填——
**填到第三篇的时候，你就会开始看见空白格子。那些空白格子就是研究问题。**

---

## 四、初步的研究缺口（供你后续验证，不要直接当结论）

从本次阅读中观察到的、值得进一步查证的方向：

- **schema 的获取成本**：能否让 LLM 自动诱导领域 schema，减少专家依赖？
- **中文垂直领域**：现有 example（medicine / supplychain / riskmining）都是 demo 级，
  缺少在中文专业语料上的系统性评测。
- **构建成本 vs 效果权衡**：0.7 版提到"轻量构建"降低 89% token 成本，
  但轻量化和效果之间的曲线没有充分公开。
- **动态知识**：知识库更新时，已建好的图如何增量维护？
- **逻辑形式的鲁棒性**：LLM 产出的 DAG 出错时会怎样？有没有兜底？（去看 `resp_verifier.py` 和重试逻辑）

> 注意：以上只是"从代码和文档里观察到的疑点"，**必须去查近两年的最新文献确认有没有人做过**。
> 直接拿来做选题会撞车。
