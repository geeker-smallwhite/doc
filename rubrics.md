# LLM/Agent 评测中的 Rubric 设计调研（2026-04-23）

## 1. 背景
你关注的问题是：当团队说“我们会从正确性、代码质量、效率、交互行为等多维度评估智能体”时，具体怎么 judge，rubric 通常长什么样。

结论先行：业界通常不是单一 rubric，而是 **多 rubric 组合**，并与自动化硬指标、线上行为指标共同构成评测体系。

## 2. 常见 Rubric 维度（行业高频）

### 2.1 解决方案质量
1. Correctness（正确性）
- 事实、逻辑、代码行为是否正确。
- 常用判定：与参考答案比对、测试用例、执行验证。

2. Completeness（完整性）
- 是否覆盖需求所有子项与边界条件。

3. Relevance / Instruction Following（相关性/指令遵循）
- 是否回答了问题本身；是否遵循格式与约束。

4. Hallucination / Groundedness / Faithfulness（幻觉/可溯源）
- 回答是否被输入上下文或检索证据支持。
- RAG 与知识问答最常用。

### 2.2 表达与可用性
5. Clarity / Readability（清晰度/可读性）
- 表达是否结构化、易理解。

6. Conciseness（简洁性）
- 是否无冗余，信息密度是否合适。

### 2.3 代码与工程
7. Code Correctness（代码正确性）
- 代码是否满足功能与测试要求。

8. Code Quality（代码质量）
- 可维护性、复杂度、命名、结构、潜在回归风险。

### 2.4 Agent 行为
9. Tool Use / Trajectory Accuracy（工具使用/轨迹质量）
- 工具选择是否正确，步骤是否连贯高效，是否偏离目标。

10. Task Completion（任务完成度）
- 是否真正完成用户任务，而非只“给建议”。

### 2.5 安全与合规
11. Safety / Security / Fairness（安全/安全性/公平性）
- 毒性、偏见、PII 泄露、提示注入、越狱等。

### 2.6 效率与用户体验
12. Efficiency（效率）
- token、时延、轮数、成本。

13. Interaction Outcomes（交互结果）
- 用户是否接受结果，是否需要追问/返工，代码是否被保留。

## 3. 线上实验如何 judge（不是只靠 rubric）
典型是三层联合：

1. 硬指标（自动）
- 任务成功率、测试通过率、回归率、耗时、token 成本。

2. 行为代理指标（线上）
- 追问率、撤销率、代码留存率、用户不满意后续请求率。

3. Rubric 评审（LLM Judge / 人工）
- 对采样会话按维度评分，并和人工金标定期校准。

常见实践：
- 离线 benchmark 做“能力判断”；
- 在线 A/B 做“用户真实收益判断”；
- 二者同时提升才算有效改进。

## 4. 不同框架中的典型 Rubric 实现

### 4.1 G-Eval（NLG）
- 采用维度化 rubric（如 coherence、consistency、fluency、relevance）+ 表单化打分。
- 强调：提示词设计、评测步骤（CoT）会影响 judge 质量。

### 4.2 FLASK（细粒度能力）
- 将能力拆为 4 大类 12 技能：
  - Logical Thinking（逻辑正确性/稳健性/效率）
  - Background Knowledge（事实性/常识）
  - Problem Handling（理解力/洞察/完整性/元认知）
  - User Alignment（简洁/可读/无害）
- 核心价值：比单一总分更可解释。

### 4.3 RAGAS（RAG场景）
- Faithfulness、Answer Relevancy、Context Precision/Recall 等。
- 重点判断“答得对”与“是否由检索证据支撑”。

### 4.4 MT-Bench / LLM Judge（对话评测）
- 常见评估因子：helpfulness、relevance、accuracy、depth、creativity、detail。
- 常见形式：单答案 1-10 分、或 A/B pairwise 对比。

### 4.5 LangSmith / OpenEvals（工程化落地）
- 预置 rubric 维度包括：
  - correctness、conciseness、hallucination、answer relevance
  - RAG groundedness/retrieval relevance/helpfulness
  - trajectory accuracy、task completion、tool selection
  - toxicity/fairness、PII leakage/prompt injection/jailbreak

## 5. 给 Coding Agent 的可落地 Rubric（建议版）
建议使用 8 维总分 100（便于发布决策）：

1. 任务完成度（20）
2. 解决正确性（20）
3. 代码质量（15）
4. 边界与稳健性（10）
5. 工具与轨迹效率（10）
6. 交互质量（10）
7. 安全与合规（10）
8. 成本效率（5）

建议门槛：
- Release Gate：
  - 任务完成度 >= 16
  - 解决正确性 >= 16
  - 安全与合规 >= 8
  - 总分 >= 80

## 6. 实施建议
1. 先少维度（3-5个）上线，再逐步扩展，避免 early overfitting。
2. 每个维度都给“正/反例锚点”，减少 judge 漂移。
3. 对发布决策使用二值或低粒度分档（Pass/Fail 或 1-3 档）更稳。
4. 定期抽样人工复核，监控 judge 与人工一致率。
5. 离线分数和线上行为指标必须联动看，避免“离线好看、线上无收益”。

## 7. 与 Cursor 场景的对应
你最初关注的 Cursor 表述可映射为：
- 解决方案正确性 -> Correctness / Task Completion
- 代码质量 -> Code Quality / 回归风险
- 效率 -> token/时延/轮数
- 交互行为 -> 追问率、代码留存率、不满意请求率

其中，公开信息通常只会披露框架与部分代理指标，详细 rubric 和权重一般不会全部公开。

## 8. 参考来源
- CursorBench: https://cursor.com/blog/cursorbench
- Cursor online A/B（含 Code Retention、Dissatisfied User Requests）: https://cursor.com/blog/semsearch
- OpenAI Graders: https://developers.openai.com/api/docs/guides/graders
- FLASK: https://ar5iv.labs.arxiv.org/html/2307.10928v4
- G-Eval: https://ar5iv.labs.arxiv.org/html/2303.16634
- MT-Bench judge prompts: https://raw.githubusercontent.com/lm-sys/FastChat/main/fastchat/llm_judge/data/judge_prompts.jsonl
- RAGAS metrics: https://docs.ragas.io/en/stable/concepts/metrics/
- LangSmith prebuilt evaluators: https://docs.langchain.com/langsmith/prebuilt-evaluators
- OpenEvals prompt list: https://raw.githubusercontent.com/langchain-ai/openevals/main/python/openevals/prompts/__init__.py
