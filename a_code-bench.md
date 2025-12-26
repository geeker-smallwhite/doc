  调研了代码领域的评测集，比较主流的评测集存在一些问题。
  - HumanEval：代码补全领域的评测集。https://arxiv.org/abs/2107.03374
    - 所有题目都是 python 实现的，编程语言单一，python 相对于其他编程语言来讲，比较简单，没有强类型约束，相当于其他强类型语言（go，java，cpp）更容易通过编译。
    - 题目整体来说相对简单，主流的模型的评测分数都是超过 90 分的，参考意义已经不大。 https://www.datalearner.com/benchmarks/humaneval
    - 生成的代码是完全独立，没有其他的依赖关系，有被主流大模型作为训练数据打榜的嫌疑，比如该评测集已经被qwen 训练过拟合。
      - https://mp.weixin.qq.com/s/eW7cU83zU_uqyNjUSQKMMw
      - https://arxiv.org/abs/2409.01790
      - https://arxiv.org/abs/2403.07974?file=2403.07974
  - HumanEval +：
    - HumanEval+：是 HumanEval 的扩展版本，可能在问题难度、覆盖场景等方面进行了拓展，以更全面地评估模型
  - MBPP ：https://arxiv.org/abs/2108.07776，
    - 相对于 humaneval 来说，难度略高一点。
  - MBPP+：也是扩展了一些任务，任务数量扩展，编程语言从 python 扩展到了其他编程语言，比如 java、cpp。
  - BigCodeBench：
    - 主要针对 python 代码，进行代码生成，主要是独立代码。
  - Python 代码能力评测：
    - https://arxiv.org/abs/2212.10481
    - https://github.com/microsoft/PyCodeGPT/tree/main/apicoder/private-eval
    - NumpyEval：https://github.com/microsoft/PyCodeGPT/tree/main/cert/pandas-numpy-eval?spm=a2c6h.13046898.publish-article.28.6f9c6ffa89GBsw
    - PandasEval：https://github.com/microsoft/PyCodeGPT/tree/main/cert/pandas-numpy-eval
  - LiveCodeBench：https://livecodebench.github.io/。从 LeetCode、AtCoder、CodeForces 三大平台的 2023 年 5 月开始的竞赛中持续收集问题，按发布日期标记，评估时仅使用模型训练截止日期后发布的问题，避免污染。目前已更新到V6版本总共800+竞赛题
    - 评测更侧重算法能力、数学逻辑能力。针对工程业务能力的评测能力很低。
  - DEVBENCH
    - https://arxiv.org/pdf/2405.19856#page=3.61
    - 主要针对 Python 代码，涉及业务逻辑来讲简单。
  - FEAbench：https://github.com/microsoft/FEA-Bench
    - 从 github 真实仓库，得到一些单步修改的任务，但是上下文信息明显不足，模型完成难度很低。
  - aider-benchmark：https://aider.chat/2024/12/21/polyglot.html#the-polyglot-benchmark
    - 主要针对评测关注模型是否能按指定“编辑格式”返回可直接应用的补丁或完整文件，包括：whole（整文件替换）、diff、diff-fenced、udiff，以及在“architect 模式”下用于二段式管线的 editor-diff/editor-whole。不同模型在不同格式下的稳定性与效率存在差异。
  - FullStackBench：https://github.com/bytedance/FullStackBench
    - 评测 LLM + agent。
  - GitTaskBench：https://arxiv.org/abs/2508.18993
    - 评测 LLM + agent。
  - SWE-bench：https://www.swebench.com，分为 Lite、FUll、Verified、mul 四个，目前主要是采用 verified 这个评测集。
    - 评测 LLM + agent。
  - SWE-Lancer：https://openai.com/index/swe-lancer/
    - 评测 LLM + agent。
上面几乎总结了目前市面上所有主流的代码生成的评测集，存在一些问题。
1. 整体来说，评测难度分布不均匀，从简单的单文件、独立函数生成的评测集，几乎直接跨越到仓库级别的 多文件、多函数的生成and 修改的任务。其中缺少单文件多函数级别的评测任务。
2. 生成的代码业务逻辑较少，即使是难度最大的 swe-bench 的相关评测集，其中主要考察的能力也是代码上下文检索能力，其中 swebench 的论文中有提到。https://openreview.net/pdf?id=VTF8yNQM66。几乎所有代码生成评测集的函数逻辑描述都是简单一句话，但是在实际场景中，用户写的 业务逻辑是比较长的。
3. 多数评测任务的编程语言都是针对 python 编程语言的，而且不能轻易拓展到其他语言。python 只占代码仓库 26% 左右。https://hellogithub.com/report/tiobe。而且在实际业务中，大多数都是 go、java、cpp 这样的编程语言，python 只占少数。
4. 多数评测集评测代码生成，仅通过单元测试是否通过来判定代码生成是否正确，只关注简单的业务逻辑，不关注生成的代码质量。
5. 代码业务逻辑比较独立，不存在跨仓库服务调用，多数业务代码可能存在数据库 CURD，线上服务请求。
