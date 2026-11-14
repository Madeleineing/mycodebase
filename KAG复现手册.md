# KAG 复现手册（v0.8.0）

面向：有远程服务器、科研经验较少的初学者。目标是**先跑通最小闭环，再谈全量复现**。

---

## 0. 前置检查（决定你能不能做）

| 检查项 | 要求 | 不满足怎么办 |
|---|---|---|
| 服务器内存 | ≥ 16GB（Neo4j 4G 堆 + Java 8G 堆） | 换机器，8GB 必然 OOM |
| Docker + Compose | 有 root / 在 docker 组 | 找管理员申请权限 |
| 端口空闲 | 8887、3306、7474、7687、9000、9001 | `netstat -tulnp` 检查 |
| Python | 3.10（官方推荐） | conda create -n kag python=3.10 |
| LLM API | DashScope(通义) + SiliconFlow(bge-m3) 的 key | 先各充 100 元试水 |
| 网络 | 能拉 Docker 镜像（阿里云 spg-registry） | 国内服务器通常没问题 |

---

## 1. 第一步：先走"产品模式"，别急着写代码

```bash
curl -sSL https://raw.githubusercontent.com/OpenSPG/openspg/refs/heads/master/dev/release/docker-compose-west.yml -o docker-compose-west.yml
docker compose -f docker-compose-west.yml up -d
docker ps   # 确认 4 个容器都 Up
```

浏览器打开 `http://<服务器IP>:8887`，默认账号 `openspg` / `openspg@kag`。

**先在网页上手动建一个知识库、传两篇文档、问几个问题。**
这一步的目的不是复现，是让你脑子里先有"KAG 到底产出了什么"的直观印象。
带着这个印象回头看代码，效率差好几倍。

> 注意：配置文件里 host_addr 默认是 127.0.0.1，远程服务器要改成实际 IP 或做 SSH 端口转发。

---

## 2. 第二步：装 KAG 客户端

```bash
conda create -n kag python=3.10 && conda activate kag
git clone https://github.com/OpenSPG/KAG.git && cd KAG
pip install -e .
```

常见坑：
- `protobuf==3.20.1` 与新版本依赖冲突 → 不要升级，保持原样
- `pip install -e .` 会拉 requirements.txt 全部依赖，建议先换国内源
- 装完执行 `knext --help` 和 `python -c "import kag"` 验证

---

## 3. 第三步：冒烟测试（2 道题，成本为零）

```bash
cd kag/open_benchmark/hotpotqa
# 编辑 kag_config.yaml，填 api_key；host_addr 改成本机地址
knext project restore --host_addr http://127.0.0.1:8887 --proj_path .
knext schema commit
cd src && python indexer.py --corpus_file ../data/sub_corpus.json
python eval.py --qa_file ../data/qa_sub.json
```

产出：`hotpotqa_res_*.json`（答案）和 `hotpotqa_metrics_*.json`（EM/F1）。

**只有 2 道题，跑几分钟就完事。跑通了，说明整条链路没问题。**

---

## 4. 第四步：小样本复现（100 题）

```bash
python indexer.py --corpus_file ../data/train_corpus.json   # 994 篇
python eval.py --qa_file ../data/qa_train.json              # 100 题
```

到这一步你就有了可以写进报告的数字。

---

## 5. 第五步：全量复现（1000 题）

```bash
python indexer.py --corpus_file ../data/corpus.json   # 9221 篇
python eval.py --qa_file ../data/qa.json              # 1000 题
```

- 构建阶段有 checkpoint（`src/ckpt`），中断可续跑，**不要从头再来**
- 并发在 `kag_config.yaml` 里调：`num_chains`（默认 16）、`num_threads_per_chain`
- 先跑 `sub` 确认 token 消耗速度，再估算全量成本

---

## 6. 代码阅读顺序（不用通读 9.8 万行）

核心机制：**YAML 里的 `type:` 字符串 ↔ 代码里的 `@XXX.register("...")`**。

1. `kag/interface/` — 所有抽象基类，先建立"有哪些角色"的整体认识
2. `kag/open_benchmark/hotpotqa/kag_config.yaml` — 一份完整配置，你的阅读主线
3. `kag/builder/default_chain.py` — 构建链怎么用 `>>` 串起来
4. `kag/builder/component/extractor/` — 10 种抽取器，对应论文的几种索引类型
5. `kag/solver/pipeline/kag_static_pipeline.py` — 静态规划推理主流程
6. `kag/solver/planner/` + `kag/solver/executor/` — 规划算子与执行算子

**可以跳过**：`knext/`（3.7 万行，只是跟服务端通信的 SDK）。

---

## 7. 避坑清单

- 文档在语雀（openspg.yuque.com），0.8 版本刚重构，**文档和代码可能不同步**，以代码为准
- `kag/open_benchmark/prqa/` 是 KAG-Thinker 相关，依赖特定模型，**初学阶段别碰**
- 遇到组件找不到（`type` 无法解析）→ 检查该类文件是否被 import，注册机制依赖模块加载
- 图里没数据 → 先去 Neo4j 浏览器（7474）手动查 `MATCH (n) RETURN count(n)`
- 报 LLM 相关错 → 90% 是 api_key 或 base_url 填错
