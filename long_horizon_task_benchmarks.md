# Long Horizon Task 评测集调研

## 一、什么是 Long Horizon Task？

Long Horizon Task（长程任务）是指需要 Agent 在较长时间跨度内，通过大量连续的交互步骤来完成的复杂任务。与传统的短程任务（如单轮问答、单步代码补全）不同，Long Horizon Task 具有以下核心特征：

- **步骤数量多**：任务需要数十到数百甚至上千步的连续操作才能完成
- **执行时间长**：从数小时到数十小时不等，远超一般的几分钟级别任务
- **需要持续推理与规划**：Agent 必须在整个执行过程中保持对全局目标的理解，进行子目标分解、路径规划和动态调整
- **部分可观测性**：Agent 通常无法一次性获取所有信息，需要主动探索和收集信息
- **记忆管理**：由于交互历史很长，Agent 需要有效管理上下文记忆，避免遗忘关键信息
- **错误恢复**：在长程执行中，错误不可避免，Agent 需要具备检测错误并重新规划的能力

Long Horizon Task 是衡量 AI Agent 是否具备真正自主能力的关键测试场景。现实世界中的许多重要任务——如大型软件开发、科学研究、商业投资决策——本质上都是 Long Horizon Task。

---

## 二、符合高标准（200+ steps / 8+ 小时）的评测集

### 1. UltraHorizon（2025，滴滴/中山大学/清华）

**论文**：UltraHorizon: Benchmarking Agent Capabilities in Ultra Long-Horizon Scenarios (arXiv: 2509.21766)

**最符合要求的评测集**。这是目前专门为"超长程"场景设计的 benchmark。

**核心信息**：
- 在最重量级设置下，Agent 轨迹平均 **200k+ tokens**，**400+ tool calls**
- 标准配置下也超过 **35k tokens**，**60+ tool calls**
- 设计了三个不同的探索环境，Agent 需要通过持续推理、规划、记忆管理和工具使用来迭代发现隐藏规则

**评测维度**：
- 持续推理能力（Sustained Reasoning）
- 规划能力（Planning）
- 记忆管理（Memory Management）
- 工具使用（Tool Use）

**关键发现**：
- LLM Agent 在长程场景中表现持续不佳
- 人类参与者得分显著高于 Agent，说明 Agent 的长程能力存在明显差距
- 简单的 scaling 在该任务上无效
- 识别出 8 种错误类型，归因于两个主要原因：**上下文锁定（in-context locking）** 和 **基础能力缺陷（functional fundamental capability gaps）**

**适用场景**：评估 Agent 在超长交互序列中的核心能力

---

### 2. RE-Bench（2024，METR）

**论文**：RE-Bench: Evaluating frontier AI R&D capabilities of language model agents against human experts (arXiv: 2411.15114)

**明确以 8 小时为时间尺度设计的 benchmark**。

**核心信息**：
- 包含 7 个具有挑战性的、开放式的 ML 研究工程环境
- 收集了 61 位人类专家的 71 次 **8 小时**尝试数据
- 对比了人类专家和 AI Agent 在 2 小时、8 小时、32 小时等不同时间预算下的表现

**评测任务示例**：
- 优化 ML 模型性能
- 编写自定义 Triton kernel
- 研究工程类开放式问题

**关键发现**：
- 在 2 小时预算下，最佳 AI Agent 得分是人类专家的 **4 倍**
- 在 8 小时预算下，人类专家勉强超过最佳 AI Agent
- 在 32 小时预算下，人类专家得分是最佳 AI Agent 的 **2 倍**
- AI Agent 生成和测试方案的速度比人类快 **10 倍以上**，成本更低
- 但人类在时间预算增加时表现出更好的收益递增

**适用场景**：评估 AI Agent 在 ML 研发任务上的长时间自主工作能力，直接对标人类专家

---

### 3. MLE-bench（2024，OpenAI）

**论文**：MLE-bench: Evaluating Machine Learning Agents on Machine Learning Engineering (arXiv: 2410.07095, ICLR 2025)

**核心信息**：
- 从 Kaggle 精选 **75 个** ML 工程竞赛任务
- 任务涵盖模型训练、数据集准备、实验运行等真实 ML 工程技能
- 使用 Kaggle 公开排行榜建立人类基线
- Agent 需要完成完整的 ML pipeline：数据探索 → 特征工程 → 模型选择 → 训练 → 调参 → 提交

**执行规模**：
- 单个竞赛任务的完整执行涉及大量步骤（数据下载、EDA、多轮模型迭代、超参搜索等）
- 复杂任务的 Agent 执行时间可达数小时
- 最佳配置（o1-preview + AIDE scaffolding）仅在 16.9% 的竞赛中达到 Kaggle 铜牌水平

**关键发现**：
- 资源 scaling（更多计算、更多尝试）对 Agent 性能有显著影响
- 即使是最强的 Agent 配置，大部分竞赛仍无法完成
- 预训练数据污染对结果有一定影响

**适用场景**：评估 Agent 在真实 ML 工程任务上的端到端能力

---

### 4. Cybench（2024，Stanford，ICLR 2025）

**论文**：Cybench: A Framework for Evaluating Cybersecurity Capabilities and Risks of Language Models (arXiv: 2408.08926)

**核心信息**：
- 包含来自 4 个 CTF 竞赛的 **40 个**专业级网络安全挑战任务
- 每个任务都在独立环境中初始化，Agent 可以执行命令并观察输出
- 引入了子任务（subtask）机制，将复杂任务分解为中间步骤

**执行规模**：
- 最简单的任务人类团队需要约 11 分钟
- **最难的任务人类团队需要 24 小时 54 分钟**
- Agent 需要进行大量的探索、漏洞分析、exploit 编写和调试
- 复杂任务涉及数百步操作

**评测模型**：GPT-4o, o1-preview, Claude 3 Opus, Claude 3.5 Sonnet, Mixtral 8x22b, Gemini 1.5 Pro, Llama 3 70B/405B

**关键发现**：
- 无子任务引导时，Agent 只能解决人类团队 11 分钟内完成的简单任务
- 对于需要数小时的复杂任务，当前 Agent 基本无法完成
- 不同 Agent scaffold 对性能有显著影响

**适用场景**：评估 Agent 在网络安全攻防场景中的长程自主操作能力

---

### 5. NL2Repo-Bench（2025，ByteDance/M-A-P）

**论文**：NL2Repo-Bench: Towards Long-Horizon Repository Generation Evaluation of Coding Agents (arXiv: 2512.12730)

**核心信息**：
- 专门评估 coding agent 的长程代码仓库生成能力
- 给定一份自然语言需求文档和空白工作区，Agent 需要自主完成：架构设计 → 依赖管理 → 多模块实现 → 生成可安装的 Python 库
- 涉及 **数百个交互步骤**

**关键发现**：
- 即使最强的 Agent，平均测试通过率也低于 **40%**，很少能完整正确地完成整个仓库
- 识别出关键的长程失败模式：
  - **过早终止（Premature Termination）**
  - **全局一致性丧失（Loss of Global Coherence）**
  - **脆弱的跨文件依赖（Fragile Cross-file Dependencies）**
  - **数百步交互中的规划不足（Inadequate Planning）**

**适用场景**：评估 Agent 从零构建完整软件项目的长程能力

---

### 6. SWE-bench（2024，Princeton）

**论文**：SWE-bench: Can Language Models Resolve Real-World GitHub Issues? (arXiv: 2310.06770)

**核心信息**：
- 包含 **2,294 个**来自真实 GitHub issue 的软件工程任务
- 变体：SWE-bench Lite（300 个）、SWE-bench Verified（500 个）、SWE-bench Live（1,319 个，持续更新）
- Agent 需要理解 issue 描述、定位相关代码、跨文件修改、通过测试

**执行规模**：
- 复杂 issue 需要跨多个文件理解和修改
- Agent 通常需要数十到上百步操作（代码搜索、文件读取、编辑、测试运行等）
- 高难度任务的执行时间可达数小时
- 但大部分单任务不到 8 小时

**适用场景**：评估 Agent 在真实软件工程场景中的 bug 修复和功能实现能力。是目前最广泛使用的 coding agent benchmark。

---

## 三、接近但未完全达标的评测集

### 7. DeepPlanning（2026，Alibaba Qwen Team）

**论文**：DeepPlanning: Benchmarking Long-Horizon Agentic Planning with Verifiable Constraints (arXiv: 2601.18137)

- 多日旅行规划和多产品购物任务
- 需要主动信息获取、局部约束推理和全局约束优化
- 强调可验证的约束满足（时间预算、财务预算等）
- 步骤数较多但单任务执行时间通常不到 8 小时
- 即使前沿 Agent 也表现挣扎

### 8. OdysseyBench（2025，Edinburgh/Microsoft）

**论文**：OdysseyBench: Evaluating LLM Agents on Long-Horizon Complex Office Application Workflows (arXiv: 2508.09124)

- 跨 Word、Excel、PDF、Email、Calendar 等办公应用的长程工作流
- 包含 300 个真实用例任务 + 302 个合成复杂任务
- 需要从长程交互历史中识别关键信息并进行多步推理
- 侧重办公场景的长程上下文依赖

---

## 四、总结对比

| 评测集 | 最大步骤数 | 最大执行时间 | 领域 | 是否达到 200+ steps / 8h+ |
|--------|-----------|-------------|------|--------------------------|
| **UltraHorizon** | 400+ tool calls | 长（200k+ tokens） | 通用探索 | ✅ 最符合 |
| **RE-Bench** | 大量（开放式） | **明确 8 小时设计** | ML 研发 | ✅ 符合 |
| **MLE-bench** | 大量（完整 ML pipeline） | 数小时 | ML 工程/Kaggle | ⚠️ 接近 |
| **Cybench** | 数百步 | 最难任务 **24h+**（人类） | 网络安全 CTF | ⚠️ 人类达标，Agent 尚无法完成 |
| **NL2Repo-Bench** | 数百步交互 | 数小时 | 代码仓库生成 | ⚠️ 接近 |
| **SWE-bench** | 数十~上百步 | 数小时（复杂任务） | 软件工程 | ⚠️ 部分任务接近 |
| DeepPlanning | 较多 | 通常 < 8h | 旅行/购物规划 | ❌ 未达标 |
| OdysseyBench | 较多 | 通常 < 8h | 办公工作流 | ❌ 未达标 |

## 五、结论

真正满足"200+ steps、8+ 小时执行时间"这一严格标准的评测集目前非常稀少：

1. **UltraHorizon** 是目前最直接针对"超长程"设计的 benchmark，400+ tool calls 明确超过 200 步要求
2. **RE-Bench** 是唯一明确以 8 小时为时间尺度设计的 benchmark，且有人类专家对照数据
3. **Cybench** 的最难任务人类需要 24+ 小时，但当前 Agent 无法完成这些任务
4. **MLE-bench** 和 **NL2Repo-Bench** 在步骤数上接近要求，但执行时间通常不到 8 小时

这反映了一个重要现实：**当前 AI Agent 的能力瓶颈恰恰在于长程任务**。大多数 benchmark 的设计上限受限于当前 Agent 的能力天花板——如果 Agent 连短程任务都做不好，设计超长程 benchmark 的意义有限。随着 Agent 能力的提升，预计会有更多真正的 long horizon benchmark 出现。
