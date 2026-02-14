# Research Tips

* When an experiment has been running for a long time without results: check the correctness of the code, run a small-scale experiment to estimate the time, or split it into multiple parallelizable experiments.
* When using PyTorch, utilize the MPS device to accelerate computations.
* Maintain a clean git repository: only commit essential content. Do not commit outdated versions, temporary test files, or intermediate log files.
* Complete tasks one by one. Add remaining tasks and items to be confirmed later to HEARTBEAT.md, clearly marking dependencies. Only when you are idle (or waiting for some tasks to finish), consider starting parallel tasks that have no dependencies and do not interfere with each other.

## Frontend Development Tips (Alpha Mining Project)

* API 引用问题：确保 JavaScript API 对象正确导出到 window 对象，且脚本加载顺序正确。api.js 必须在 backtest.js 和 portfolio.js 之前加载。
* 表单验证：使用 HTML5 内置验证时，浏览器默认错误消息不够清晰。应添加自定义验证函数，提供具体的错误提示（如："学习率必须在 0.001-1 之间"）。
* 日期控件样式：input[type="date"] 需要在 CSS 中单独指定样式，使其与其他输入框保持一致。
* 图标选择：优先使用 Font Awesome 图标而非 emoji，提供更专业、统一的外观。

## Backend API Testing Tips (Alpha Mining Project)

* Qlib instruments API：D.instruments(market='csi300') 在某些版本可能返回错误格式。备选方案是直接从 instruments 文件读取股票列表（如 ~/.qlib/qlib_data/cn_data/instruments/csi300.txt）。
* 数据库架构迁移：当数据库模型变更时，最简单的方法是删除旧数据库文件重新创建。长期应实现数据库迁移机制。
* API 参数设计：FastAPI 中，使用 Query() 声明 URL 查询参数，使用 Body() 或直接类型声明声明请求体参数。注意区分 GET 和 POST 请求的参数传递方式。
* 测试分离：直接测试业务逻辑（如 MyAlpha handler）和通过 API 测试是两个层面。直接测试可以快速定位问题，API 测试验证端到端功能。

## Deep Learning Factor Model Optimization Insights (2026-02-14)

* 锦标赛方法对比（LGBM模型，5种方法）：
  - **Single Elimination (单淘汰赛)** 最佳：年化收益率 15.24%，Sharpe 0.6787
  - Random Matching 第2：年化收益率 14.35%，但最大回撤最小 (-25.04%)
  - Hybrid、Double Elimination、Adaptive 效果相近（12-13%）
  - 结论：使用 Single Elimination 作为后续DL实验的排序方法
* Bug修复经验：sorting_algorithms.py:716 - 接受"random"和"random_matching"两种命名，提高配置兼容性

* 特征预处理方法对比（基线DL模型，3种方法）：
  - **Baseline（无预处理）** 最佳：年化收益率 17.08%，Sharpe 0.744，Calmar 0.801
  - Winsorize+Standardize 第2：年化收益率 15.42%，但IC最高（2.11%）
  - Standardize 表现最差：年化收益率 12.20%，Sharpe 0.527
  - 关键洞察：预处理并非总是有益，原始特征可能保留更多信息；IC与回测性能并非完全一致
  - 结论：后续实验使用Baseline配置（无预处理）

* 网络架构探索（10种架构对比）：
  - **Width_3x512 (3层×512神经元)** 最佳：年化收益率 16.42%，Sharpe 0.739，参数量 765,953
  - Control_2x64 (对照组): 年化收益率 16.01%，Sharpe 0.687，参数量 34,241 (性价比高)
  - Depth_3x64: 年化收益率 14.74%，Sharpe 0.636，参数量 38,401 (浅层模型表现不错)
  - 关键发现：
    - 宽度比深度更重要：3层×512显著优于6层×64
    - 参数量与性能非严格正相关：Width_3x512性能接近但未超过Baseline
    - 过深网络性能下降：4层和6层性能显著下降（可能梯度消失/过拟合）
    - 瓶颈和纺锤结构表现中等：Bottleneck 11.99%，Spindle 12.70%
    - 双塔结构表现较差：10.10%，可能是特征融合方式不当
  - 结论：后续实验使用Width_3x512作为架构，但需考虑计算成本；Control_2x64是轻量级备选

## Background Task Management Tips (2026-02-14)

* Claude Code 任务可能进入交互模式卡住，导致任务无法继续执行
* 解决方案：在任务文件末尾添加明确的退出指令，避免进入交互模式
* 任务文件必须包含：
  ```text
  当完全完成时，运行此命令通知我：
  openclaw gateway wake --text "任务完成信息" --mode now
  ```
* 对于长时间运行的实验，添加分阶段输出指令：
  ```text
  对于长时间运行的实验，分阶段执行。在每个阶段之后，保存并报告中间结果（例如指标、检查点或日志），然后再进行下一阶段。不要等到整个实验完成才提供更新
  ```
* 使用 process 工具监控任务状态：`process log <sessionId>`
* 定期检查代码仓库中的结果文件，即使Claude没有直接通知你