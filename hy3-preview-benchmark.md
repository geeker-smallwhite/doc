# Hy3 preview 模型报告 Benchmark 调研整理

整理日期：2026-04-23  
报告来源：[腾讯混元 HY3 研究页](https://hy.tencent.com/research/HY3)；页面配置数据来自 `commonAssets/hy3/hy3-config-zh.json`，附录 benchmark 表来自 `conclusion.jpg`。  
范围说明：下表整理报告附录中出现的 30 个 benchmark。公开来源优先采用论文、官网、GitHub、Hugging Face；未找到稳定公开来源或疑似内部集的条目已标注。

## 总览

| 类别 | Benchmark | 报告中名称/别名 | 公开资源 | 备注 |
|---|---|---|---|---|
| Reasoning | GPQA-Diamond | GPQA-Diamond | [Paper: arXiv 2311.12022](https://arxiv.org/abs/2311.12022) / [GitHub](https://github.com/idavidrein/gpqa) / [HF Dataset](https://huggingface.co/datasets/Idavidrein/gpqa) | Graduate-level Google-proof Q&A；Diamond 是 GPQA 高质量子集。 |
| Reasoning | Humanity's Last Exam | HLE | [Paper: arXiv 2501.14249](https://arxiv.org/abs/2501.14249) / [GitHub](https://github.com/centerforaisafety/hle) / [HF Dataset](https://huggingface.co/datasets/cais/hle) | CAIS 等发布的高难综合知识与推理评测。 |
| Reasoning | PHYBench | PHYBench | [Paper: arXiv 2504.16074](https://arxiv.org/abs/2504.16074) / [Project](https://phybench-official.github.io/phybench-demo/) | 物理感知与推理评测，覆盖高中到竞赛/本科物理题。 |
| Reasoning | FrontierScience | FrontierSci-Olympiad | [OpenAI page](https://openai.com/index/frontierscience/) | OpenAI 发布的科学研究能力 benchmark；Hy3 表中使用 Olympiad tier。未发现独立公开数据仓库。 |
| Reasoning | IMO-AnswerBench | IMOAnswerBench | [LLMDB entry](https://llmdb.com/benchmarks/imo-answerbench) | 公开资料显示为 Google DeepMind IMO-Bench 系列的一部分；未找到可直接确认的官方 GitHub/HF 数据集。 |
| Reasoning | ARC-AGI-1 | ARC-AGI-1 | [Paper: arXiv 1911.01547](https://arxiv.org/abs/1911.01547) / [GitHub](https://github.com/fchollet/ARC-AGI) / [ARC Prize](https://arcprize.org/arc-agi) | Abstraction and Reasoning Corpus，常用于抽象推理/通用智能评测。 |
| Reasoning | Tsinghua Math PhD Qual, Spring 2026 | Tsinghua Math PhD Qual | 未找到稳定公开 benchmark 仓库 | 更像报告自建的近期博士资格考试题集合；需要以报告口径为准。 |
| Reasoning | Princeton Physics PhD Qual, Jan 2026 | Princeton Phys PhD Qual | 未找到稳定公开 benchmark 仓库 | 更像报告自建的近期博士资格考试题集合；需要以报告口径为准。 |
| Reasoning | China High School Biology Olympiad 2025 | CHSBO25 / China HS Bio Olympiad | 未找到稳定公开 benchmark 仓库 | 更像竞赛题集合；未发现独立 AI benchmark 项目页。 |
| Agentic Coding | SWE-bench Multilingual / Multi-SWE-bench | SWE-bench Multi. | [SWE-bench Multilingual](https://www.swebench.com/multilingual-leaderboard.html) / [Multi-SWE-bench HF](https://huggingface.co/datasets/bytedance-research/Multi-SWE-Bench) | 报告表中写作 “SWE-bench Multi.”；公开生态中存在 SWE-bench Multilingual 与 ByteDance Multi-SWE-bench 两个相近名称，需按报告原始配置进一步核验。 |
| Agentic Coding | SWE-bench Verified | SWE-bench Verified | [Official](https://www.swebench.com/) / [Paper: arXiv 2310.06770](https://arxiv.org/abs/2310.06770) / [GitHub](https://github.com/SWE-bench/SWE-bench) / [HF Dataset](https://huggingface.co/datasets/princeton-nlp/SWE-bench_Verified) | 真实 GitHub issue 修复任务，Verified 为人工筛选子集。 |
| Agentic Coding | SWE-bench Pro | SWE-bench Pro | [HF Dataset](https://huggingface.co/datasets/ScaleAI/SWE-bench_Pro) / [GitHub](https://github.com/scaleapi/SWE-bench_Pro-os) / [Paper PDF](https://static.scale.com/uploads/654197dc94d34f66c0f5184e/SWEAP_Eval_Scale%20(9).pdf) | Scale AI 发布的更长链路、企业级 SWE 任务集。 |
| Agentic Coding | Terminal-Bench 2.0 | Terminal-Bench 2.0 | [Official](https://www.tbench.ai/) / [GitHub](https://github.com/harbor-framework/terminal-bench) / [Paper: arXiv 2601.11868](https://arxiv.org/abs/2601.11868) | 真实终端环境中的长链路 agent 任务。 |
| Agentic Coding | Hy-Backend | Hy-Backend | 未公开 | Hy3 报告中的混元内部后端工程 benchmark。 |
| Agentic Coding | Hy-Vibe | Hy-Vibe | 未公开 | Hy3 报告中的混元内部 vibe coding benchmark。 |
| Agentic Coding | Hy-SWE Max | Hy-SWE Max | 未公开 | Hy3 报告中的混元内部 SWE benchmark。 |
| Agentic Search | BrowseComp | BrowseComp | [OpenAI page](https://openai.com/index/browsecomp/) / [Paper: arXiv 2504.12516](https://arxiv.org/abs/2504.12516) / [GitHub: simple-evals](https://github.com/openai/simple-evals) | OpenAI 浏览检索 benchmark，测 hard-to-find short-answer browsing。 |
| Agentic Search | WideSearch | WideSearch | [Project](https://widesearch-seed.github.io/) / [Paper: arXiv 2508.07999](https://arxiv.org/abs/2508.07999) / [GitHub](https://github.com/ByteDance-Seed/WideSearch) | ByteDance Seed 发布，面向大规模信息搜集可靠性。 |
| Agentic Search | FinSearchComp | FinSearchComp | [Project](https://randomtutu.github.io/FinSearchComp/) / [Paper: arXiv 2509.13160](https://arxiv.org/abs/2509.13160) / [HF Dataset](https://huggingface.co/datasets/ByteSeedXpert/FinSearchComp) | 金融搜索与推理 benchmark，包含现实金融分析工作流。 |
| Agentic Search | SEAL-0-260320 | SEAL-0-260320 / Seal-O-260320 | 未找到稳定公开来源 | 报告中出现的搜索/agent 评测名称；公开搜索未确认官方项目。 |
| Tool Use & Claw | tau2-bench | Tau2-bench / τ²-bench | [GitHub](https://github.com/sierra-research/tau2-bench) / [Paper: arXiv 2506.07982](https://arxiv.org/abs/2506.07982) | 双控制环境下的对话 agent/tool use 评测；同仓库也包含 τ-bench/τ³-bench 相关版本。 |
| Tool Use & Claw | WebArena | WebArena | [Official](https://webarena.dev/) / [Paper: arXiv 2307.13854](https://arxiv.org/abs/2307.13854) / [GitHub](https://github.com/web-arena-x/webarena) | 真实网站环境中的 web agent benchmark。 |
| Tool Use & Claw | ClawEval-260325 | ClawEval-260325 | [候选 GitHub: AIgenteur/ClawEval](https://github.com/AIgenteur/ClawEval) | 仅找到疑似相关仓库，未能确认与报告版本号 `260325` 完全对应；建议按腾讯报告配置二次确认。 |
| Tool Use & Claw | WildClawBench | WildClawBench | [GitHub](https://github.com/InternLM/WildClawBench) / [Project](https://internlm.github.io/WildClawBench/) | OpenClaw 环境中的 in-the-wild agent benchmark。 |
| Tool Use & Claw | SkillsBench | SkillsBench | [Official](https://www.skillsbench.ai/) / [Paper: arXiv 2602.12670](https://arxiv.org/abs/2602.12670) / [GitHub](https://github.com/benchflow-ai/skillsbench) | 评估 agent 使用技能包的效果与技能组合能力。 |
| Instruction Following & Long Context | AdvancedIF | AdvancedIF | [GitHub](https://github.com/facebookresearch/AdvancedIF) / [HF Dataset](https://huggingface.co/datasets/meta-llama/AdvancedIF) | Meta/Facebook Research 发布的高级指令遵循 benchmark。 |
| Instruction Following & Long Context | AA-LCR | AA-LCR | [Artificial Analysis announcement](https://artificialanalysis.ai/articles/announcing-aa-lcr) / [AI Wiki](https://aiwiki.ai/wiki/aa-lcr) | Artificial Analysis Long Context Reasoning；未发现公开数据仓库。 |
| Instruction Following & Long Context | LongBench v2 | LongBench v2 | [Project](https://longbench2.github.io/) / [Paper: arXiv 2412.15204](https://arxiv.org/abs/2412.15204) / [GitHub](https://github.com/THUDM/LongBench) / [HF Dataset](https://huggingface.co/datasets/THUDM/LongBench-v2) | 长上下文深度理解与推理评测，503 道多选任务。 |
| Instruction Following & Long Context | CL-bench | CLBench / CL-bench | [Leaderboard](https://www.clbench.com/) / [Paper: arXiv 2602.03587](https://arxiv.org/abs/2602.03587) / [GitHub](https://github.com/Tencent-Hunyuan/CL-bench) / [HF Dataset](https://huggingface.co/datasets/tencent/CL-bench) | 腾讯混元 Context Learning benchmark，评估模型从上下文学习新知识的能力。 |
| Instruction Following & Long Context | CL-bench Life | CLBench-life / CL-bench life | [CL-bench GitHub](https://github.com/Tencent-Hunyuan/CL-bench) / [CL-bench HF](https://huggingface.co/datasets/tencent/CL-bench) | 报告中列为 CL-bench 的 Life 子集/场景；未找到独立公开仓库，可能在 CL-bench 数据/配置中体现。 |

## 按可核验程度分类

### 公开且资源较完整

GPQA-Diamond、HLE、PHYBench、ARC-AGI-1、SWE-bench Verified、SWE-bench Pro、Terminal-Bench 2.0、BrowseComp、WideSearch、FinSearchComp、τ²-bench、WebArena、WildClawBench、SkillsBench、AdvancedIF、LongBench v2、CL-bench。

### 有公开说明，但数据或仓库不完全公开

FrontierScience / FrontierSci-Olympiad、IMO-AnswerBench、AA-LCR、CL-bench Life。

### 报告引用或疑似内部集，未找到稳定公开地址

Tsinghua Math PhD Qual Spring 2026、Princeton Physics PhD Qual Jan 2026、CHSBO25、Hy-Backend、Hy-Vibe、Hy-SWE Max、SEAL-0-260320、ClawEval-260325。

## 需要二次确认的点

| 条目 | 不确定点 | 建议 |
|---|---|---|
| SWE-bench Multi. | 可能指 SWE-bench Multilingual，也可能指 ByteDance Multi-SWE-bench；两者均为多语言 SWE 方向，但样本规模/来源不同。 | 若要复现实验，需要拿到 HY3 eval 配置中的 dataset id 或 harness 名称。 |
| ClawEval-260325 | 搜到疑似 `AIgenteur/ClawEval` 仓库，但无法确认是否为报告版本。 | 以 Tencent 内部 benchmark registry 或报告作者口径为准。 |
| SEAL-0-260320 | 未找到可核验公开页面。 | 可能是内部或特定日期快照评测，需要报告配置。 |
| 近期考试题集合 | 清华数学博资、普林物理博资、CHSBO25 更像按时间截取的考试/竞赛题集合。 | 若需要复现，需要题源、题号范围、评分脚本和防污染策略。 |

## 参考链接

- HY3 报告页：https://hy.tencent.com/research/HY3
- HY3 页面配置：https://hunyuan-portal-prod-1258344703.cos.ap-guangzhou.myqcloud.com/commonAssets/hy3/hy3-config-zh.json
- HY3 附录图：https://hunyuan-portal-prod-1258344703.cos.ap-guangzhou.myqcloud.com/commonAssets/hy3/conclusion.jpg
