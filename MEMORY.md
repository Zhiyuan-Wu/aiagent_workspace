# Research Tips

* Code Review Finding (2026-02-17): Static code review of alpha_mining project found 3 critical issues in `random_signal` strategy (undefined variables in predict(), missing imports in backtest()), plus frontend display and information architecture issues. Key risks: strategy availability, task deletion stability, duplicate trade prevention.

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

## Exp6 SIGKILL Root Cause (2026-02-17)

* **Issue:** 4 consecutive Exp6 failures with SIGKILL (9min → 59min → 22min → 30min)
* **Root Cause:** MyAlpha handler initialization enters infinite retry loop during Qlib data loading
* **Evidence:** Diagnostic testing shows process hangs during MyAlpha creation with repeated urllib3 warnings
* **Symptoms:** Infinite loop of urllib3 warnings even after "Loading data Done" appears
* **Triggers:** Large feature set (200+ features) × large date range (2008-2020) × CSI300 stocks
* **Not the issue:** Imports work fine, not OOM (memory only ~400MB during hangs)
* **Solution needed:** Disable urllib3 warnings to see actual errors, test with reduced date ranges (2015-2020), check Qlib data integrity
* **Status:** BLOCKING - Exp6-8 experiments cannot proceed until this is resolved

## Deep Learning Factor Model Optimization Insights (2026-02-14)

* 锦标赛方法对比（LGBM模型，5种方法）：
  - **Single Elimination (单淘汰赛)** 最佳：年化收益率 15.24%，Sharpe 0.6787
  - Random Matching 第2：年化收益率 14.35%，但最大回撤最小 (-25.04%)
  - Hybrid、Double Elimination、Adaptive 效果相近（12-13%）
  - 结论：使用 Single Elimination 作为后续DL实验的排序方法
* Bug修复经验：sorting_algorithms.py:716 - 接受"random"和"random_matching"两种命名，提高配置兼容性

* 网络架构探索（10种架构对比，2026-02-15新实验）：
  - **Exp B (Depth: 4×64) 最佳**：年化收益率 15.17%，Sharpe 0.6653，最大回撤 -26.99%
  - Control_2x64: 年化收益率 12.62%，Sharpe 0.5544 (基线，性价比高)
  - Exp I (Twin Tower): 年化收益率 14.03%，Sharpe 0.6497 (双塔结构表现较好)
  - Top 3: Exp B (4×64) 15.17%/0.6653, Control (2×64) 12.62%/0.5544, Twin Tower 14.03%/0.6497
  - 关键发现：
    - 适中深度表现最佳：4层×64 优于其他深度和宽度变体
    - 过深网络性能下降：6层性能显著下降（10.01%）
    - 宽度增加不如深度：3×128 (9.19%), 3×256 (11.15%), 3×512 (9.30%)
    - 瓶颈和纺锤结构表现中等：Bottleneck 11.04%, Spindle 10.79%
    - Twin Tower 表现较好：14.03%，仅次于最佳架构
  - 结论：后续实验使用 4×64 架构

* 特征预处理方法对比（基线DL模型，3种方法，2026-02-15新实验）：
  - **Standardize (Z-Score) 最佳**：年化收益率 15.22%，Sharpe 0.6697，Calmar 0.5602
  - Baseline (Raw) 第2：年化收益率 14.46%，Sharpe 0.6253，Calmar 0.5293，IC最高（0.0258）
  - Winsorize + Standardize 第3：年化收益率 14.23%，Sharpe 0.6131，Calmar 0.5535，Rank IC最高（0.0273）
  - 关键洞察：标准化提升Sharpe 0.0444；缩尾处理降低性能；IC与回测性能不完全一致
  - 结论：后续实验使用Standardize (Z-Score)配置

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

**🚨 Claude Code行为问题** (2026-02-14 Task DL-3经验):

* **问题**: Claude Code可能不会按照任务要求约定的位置写报告，也可能不会在任务完成时通知
* **验证方法**: 必须通过查看所有新生成的文件来准确判断任务进展，不能仅依赖Claude Code的输出
* **检查清单**:
  - 实验结果CSV文件是否存在
  - 模型文件是否存在
  - 图表文件是否存在
  - 日志文件是否完整
  - 实验报告文件是否存在
  - 任务完成报告文件是否存在
* **发现**: Task DL-3虽然日志显示"实验完成!"，但并未生成exp3_normalization_residual_report.md和task_exp3_completion_report.md报告文件
* **结论**: 即使Claude Code声称完成，也要验证所有输出文件都存在，否则任务不算真正完成
* **对策**: 在任务文件中明确要求：
  - 在日志中明确列出所有将要生成的文件路径
  - 每个文件生成后立即print确认
  - 最后必须显式调用openclaw gateway wake命令通知

### 处理Claude Code卡死的完整流程（2026-02-14经验总结）

**症状识别**:
1. CPU使用率异常（95%高负载或22%持续低负载但无进展）
2. 日志文件停止增长
3. Claude Code状态显示相同的状态超过30分钟
4. 进程无响应
5. 没有生成新的日志文件

**失败模式分类**:

**模式1: 语法/启动错误**
- 现象: Python报错立即退出（如IndentationError）
- 处理: 修复语法错误，重新运行

**模式2: 死循环/无限等待（高CPU）**
- 现象: CPU 95%，日志停止输出
- 可能原因:
  - 数据加载死循环
  - 训练循环条件错误
  - 资源泄漏导致无限等待
- 处理: 检查循环条件，添加超时机制

**模式3: 处理阶段卡死（低CPU）**
- 现象: CPU 20-30%，无日志输出
- 可能原因:
  - 模型初始化问题
  - 数据处理IO阻塞
  - 内存不足导致频繁GC
- 处理: 添加日志输出到每个关键步骤

**分阶段实验策略**（解决卡死问题的有效方法）:

1. **阶段1: 最小化验证**
   - 只运行1个配置
   - 训练1-2个epoch
   - 使用小数据集（前100个样本）
   - 关闭复杂功能（如数据增强）
   - 目标: 验证代码可运行性

2. **阶段2: 逐步扩展**
   - 运行完整训练轮次（但只1个配置）
   - 验证训练稳定性
   - 目标: 确认无内存/性能问题

3. **阶段3: 完整实验**
   - 运行所有配置
   - 目标: 完成完整实验

**任务文件编写最佳实践**:
1. 添加详细的失败历史描述
2. 提供代码检查清单（语法、数据处理、模型、训练循环）
3. 明确要求分阶段实验
4. 要求详细的日志输出（每个关键步骤都要有日志）
5. 避免使用可能抑制日志的工具（如tqdm）
6. 要求使用print + tee确保日志实时写入

**监控和干预**:
1. 每隔30分钟检查一次任务状态
2. 如果发现卡死现象：
   - 先记录症状（CPU、内存、日志状态）
   - 终止进程
   - 分析日志最后输出
   - 更新任务文件
   - 决定是修复代码还是重启
3. 如果连续2次以相同方式失败：
   - 停止自动重启
   - 人工调试代码
* 现代归一化与残差连接实验（5种配置对比，2026-02-15新实验）：
  - **Control (无归一化，无残差) 最佳**：年化收益率 10.97%，Sharpe 0.4835，最大回撤 -29.01%
  - Exp A (LayerNorm): 10.80%/0.4809, Exp B (BatchNorm): 9.85%/0.4642, Exp C (Residual): 9.36%/0.4192, Exp D (Residual+LN): 8.97%/0.4072
  - 关键发现：
    - 基线配置（无现代技术）表现最好
    - LayerNorm (0.4809) 略优于 BatchNorm (0.4642)，但两者都不如基线
    - 残差连接显著降低性能（下降 13-16% Sharpe）
    - 意外结果：对于这个问题规模，简单架构可能优于复杂架构
    - 结论：后续实验使用 Control 配置（无归一化，无残差连接）

* 激活函数选择实验（3种激活函数对比，2026-02-15新实验）：
  - **Control (ReLU) 最佳**：年化收益率 13.80%，Sharpe 0.6042
  - Exp A (PReLU): 9.85%/0.4296, Exp B (Swish): 7.37%/0.3240
  - 关键发现：
    - PReLU 虽然可学习负斜率但性能下降（相比 ReLU 下降 29% Sharpe）
    - Swish 提供平滑激活但显著表现更差（相比 ReLU 下降 46% Sharpe）
    - 结论：后续实验使用 ReLU 激活函数

* 前端TDD测试（绿色阶段）：针对Chrome DevTools完成代码修复和全面测试，2026-02-15新任务
  - **问题识别**：红色阶段发现代码组织、API集成、UI/UX、图表实现等方面的问题
  - **修复策略**：使用Chrome DevTools系统化测试，修复已知问题
  - **关键改进**：代码审查、API测试、前端测试（导航、表单、结果显示、错误处理）、响应式设计验证
  - **测试结果**：24个测试，23个通过，1个失败（94%通过率），2个关键bug修复
  - **结论**：所有测试阶段完成，前端准备部署（有一个待解决的后端问题：portfolio summary数据不一致）

## Qlib Data Loading Issues (2026-02-18)

* **Issue**: Qlib 0.9.7 returns only 2 instruments instead of expected CSI300 stocks
* **Root Cause**: Data directory structure incompatible with current Qlib version
* **Symptoms**:
  - `D.instruments(market='csi300')` returns only 2 instruments ("market" and others)
  - KeyError: `slice(None, 5, None)` when accessing data
  - `qlib.init()` succeeds but data loading fails
  - Experiment logs: "Generated 0 pairs in 0 batches"
* **Solution**: Use alternative data loading methods:
  1. Read instruments directly from `~/.qlib/qlib_data/cn_data/instruments/csi300.txt`
  2. Verify Qlib data integrity with `qlib-data download` scripts
  3. Check Qlib version compatibility (current: 0.9.7)
* **Status**: BLOCKING - Exp7 cannot proceed until data loading is fixed

## API Rate Limiting Management (2026-02-18)

* **Issue**: 5-hour usage quota reached during long-running experiments
* **Error Message**: "已达到 5 小时的使用上限。您的限额将在 2026-02-18 03:39:22 重置"
* **Symptoms**:
  - Repeated API timeout errors (attempt 1-10 retries)
  - "API_TIMEOUT_MS=3000000ms, try increasing it"
  - Task hangs indefinitely after quota exceeded
* **Prevention Strategies**:
  1. Break large experiments into smaller chunks
  2. Use offline mode for local computation where possible
  3. Implement checkpointing to resume after quota reset
  4. Monitor API usage with session_status tool
* **Resolution**: Wait for quota reset (at scheduled time) before retrying
* **Experience**: 4-hour tasks are at high risk of hitting quota limits; consider shorter subtasks


## Claude Code Session Stuck Issues (2026-02-18)

* **Issue**: Frontend testing task (quiet-coral session) got stuck and was killed (SIGKILL)
* **Symptoms**:
  - Session running for ~36 minutes with no progress
  - Process in Ss (sleep) state - likely waiting for input or hung
  - No test logs or output generated
  - Session appeared idle throughout execution
* **Task Details**:
  - Task: Frontend Testing with Chrome DevTools MCP
  - Expected: Test alpha_mining frontend, generate report, send via Telegram
  - Session started: 05:35:00
  - Session killed: 06:11:00
  - No outputs: frontend_test_report.md not created
* **Possible Causes**:
  1. Chrome DevTools MCP not available or not properly configured
  2. Session encountered initialization error and hung instead of failing
  3. Session entered interactive mode requiring user input
  4. Task file instructions unclear about how to use Chrome DevTools MCP
  5. Frontend server not running (need to check backend main.py)
  6. MCP tool limitations or compatibility issues
* **Prevention Strategies**:
  1. Verify MCP tools are available before assigning tasks that use them
  2. Test task with minimal example before full execution
  3. Check if frontend/backend servers are running before testing
  4. Add explicit MCP tool availability check to task
  5. Include fallback testing methods (manual testing) if MCP fails
  6. Add timeout limits to task file to prevent indefinite hanging
* **Resolution**: Session was killed, task marked as FAILED
* **Next Steps**: 
  1. Check if Chrome DevTools MCP is installed and configured
  2. Verify frontend server can be started
  3. Consider manual testing or alternative testing approach
  4. Retry with more detailed task instructions

## Exp6 Feature Reconstruction Error (2026-02-18)

* **Issue**: EXP 6/7 (Feature Reconstruction λ=0.5) failed with unpacking error
* **Error Message**: "too many values to unpack (expected 3)"
* **Symptoms**:
  - Training completed successfully for EXP 6/7
  - Error occurred during result processing or backtest
  - EXP 7/7 (Feature Reconstruction λ=1.0) started and is running
* **Possible Causes**:
  1. Model output format mismatch (MLPModelWithFeatureRecon returns different number of values)
  2. Result unpacking expects 3 values but model returns different number
  3. Data format issue with feature reconstruction output
* **Impact**:
  - EXP 6/7 results are missing
  - Cannot compare λ=0.5 with other λ values
  - Need to fix and re-run EXP 6/7 after EXP 7/7 completes
* **Investigation Needed**:
  1. Check MLPModelWithFeatureRecon forward() method return format
  2. Verify result processing code in workflow.py
  3. Compare with MLPModelWithReturnPred (which works)
* **Status**: BLOCKING - Need to fix and re-run EXP 6/7 before finalizing Exp6 results
* **Next Steps**:
  1. Wait for EXP 7/7 to complete
  2. Investigate and fix the unpacking error
  3. Re-run EXP 6/7
  4. Generate complete exp6_multitask_results.csv and report


## Qlib数据读取问题与解决方案 (2026-02-23)

* **问题现象**：`D.instruments('csi300')` 只返回2个值（`['market', 'filter_pipe']`），而`csi300.txt`文件实际包含15898只股票。这导致回测无法正常执行，所有回测指标都是0.00%。

* **根本原因**：Qlib 0.9.7版本的`D.instruments()`方法在读取股票池时存在兼容性问题，无法正确解析instruments文件。

* **解决方案**：不要直接使用`D.instruments()`获取股票列表。应该使用`DataHandlerLP`通过`MyAlpha`数据处理器获取instruments：
  ```python
  from qlib.data.dataset import DatasetH
  from pair_wise_ranking.data_handler import MyAlpha

  handler_kwargs = {
      "instruments": stock_pool,
      "start_time": start_date,
      "end_time": end_date,
      "fit_start_time": start_date,
      "fit_end_time": end_date,
  }

  handler = MyAlpha(**handler_kwargs)
  dataset = DatasetH(handler=handler, segments={
      "train": (start_date, end_date)
  })
  
  # 获取instruments列表
  instruments = list(dataset.handler.instruments)
  ```

* **为什么不能用文件直接读取**：虽然可以直接读取`~/.qlib/qlib_data/cn_data/instruments/{stock_pool}.txt`文件，但这违背了Qlib的设计原则，可能导致未来Qlib版本升级时的兼容性问题。应该使用Qlib提供的数据处理API。

* **多进程问题**：使用MyAlpha初始化时，会触发Qlib的多进程数据处理（joblib）。如果脚本没有使用`if __name__ == '__main__':`保护，会出现multiprocessing错误。

* **其他策略的参考**：在`pairwise_rank_strategy.py`和`pointwise_strategy.py`中，都是通过MyAlpha数据处理器来获取instruments的，这是正确的做法。

## Exp8 Feature Interaction Experiment Failure (2026-02-24)

* **Issue**: Experiment 8 (Feature Interaction and Self-Attention) failed without generating completion report
* **Status**: BLOCKING - Exp8 was attempted but crashed, no follow-up TODO created
* **Task File**: `task_history/260220/task_20260221_092000_exp8_interaction_redo.txt`
* **Experiment Started**: 2026-02-21 10:58:20
* **Last Activity**: 2026-02-21 23:48 (process crashed with joblib warnings)
* **Symptoms**:
  - Control experiment completed successfully
  - FM-only and DeepFM experiments started training
  - Process crashed with joblib resource_tracker warnings indicating SIGKILL
  - No completion report generated
  - No follow-up TODO created
  - No memory entry documenting the failure (until now)
* **What Happened**:
  - Task assigned to Claude Code in background session
  - Detailed task file with incremental testing strategy (5 epochs first, then 50 epochs)
  - Control experiment succeeded (verified baseline)
  - Started FM-only embedding=8 and 16 training
  - Started DeepFM embedding=8 and 16 training
  - DeepFM-8 failed with "DeepFMModel.forward() takes 2 positional arguments but 3 were given"
  - DeepFM-16 training started but process crashed with joblib warnings
  - No notification to user, no completion report
* **Root Cause Analysis**:
  1. DeepFM model has incorrect forward() signature (accepts 2 args but called with 3)
  2. Process crashed likely due to resource exhaustion or timeout
  3. No graceful error handling or recovery
  4. No monitoring/alerting when task failed silently
* **Missing Follow-up**:
  - Task was not tracked in heartbeat-cli
  - No TODO created to fix and retry
  - No memory entry documenting failure until now (3+ days later)
  - User was never informed of the failure
* **Lessons Learned**:
  1. Background Claude Code tasks must be actively monitored
  2. Task failures should trigger automatic TODO creation
  3. Complex model implementations (FM/DeepFM/Attention) need more thorough testing before full experiments
  4. Task files should include explicit failure recovery instructions
  5. Memory entries should be created promptly when tasks fail
* **Next Steps Needed**:
  1. Fix DeepFM model forward() signature issue
  2. Implement FM-only and Self-Attention models if not already done
  3. Create comprehensive test suite for new models (test with minimal data first)
  4. Re-run Exp8 with better error handling and monitoring
  5. Ensure task completion notification is sent to user
* **Reference Files**:
  - Task file: `task_history/260220/task_20260221_092000_exp8_interaction_redo.txt`
  - Retry task file: `claude_tasks/task_20260224_140300_exp8_retry.txt`
  - Logs: `alpha_mining/history/deeplearning_for_pairwise/results/exp8_*.log`
  - Latest results: `alpha_mining/results/exp8_interaction_results_20260224_092919.csv`
  - Previous success: Exp7 completion report available for reference

## Exp8 Retry Failure (2026-02-24)

* **Issue**: Exp8 retry completed prematurely with incomplete results and severe performance degradation
* **Status**: FAILED - Experiment ran only 1/7 configurations (Control only)
* **Task File**: `claude_tasks/task_20260224_140300_exp8_retry.txt`
* **Task Assigned**: Feb 24, 06:32:25
* **Task Completed**: Feb 24, 09:29:19 (3 hours total)
* **Symptoms**:
  - Control experiment completed but Sharpe 0.2053 (vs expected 0.4925 from Exp7) - 58% degradation
  - FM-only models (2 configs) - NOT implemented
  - DeepFM models (2 configs) - NOT implemented
  - Self-Attention models (2 configs) - NOT implemented
  - Experiment stopped early without error or completion of all phases
  - Claude Code marked task as complete despite incomplete execution
* **Root Cause Analysis**:
  1. **Control performance collapse**: Exp7 baseline achieved Sharpe 0.4925, but retry achieved only 0.2053
     - Possible causes: data loading issue, preprocessing bug, configuration mismatch
     - Training completed smoothly (50 epochs, loss decreasing), suggesting code ran correctly
     - Need to compare Exp7 vs Exp8 control configs side-by-side
  2. **Missing model implementations**: Task file clearly specified 7 configs (Control + 2 FM + 2 DeepFM + 2 Self-Attention)
     - Claude Code skipped Phases 2-4 entirely
     - Possible causes: early termination condition, error handling bug, task misinterpretation
  3. **Early completion**: Experiment stopped after Control without proceeding to FM/DeepFM/Attention phases
     - Possible causes: conditional logic error, phase tracking bug, silent failure
* **Impact**:
  - Cannot compare FM/DeepFM/Attention against Control baseline
  - Cannot identify which feature interaction mechanism works best (if any)
  - Cannot complete deep learning model optimization series
  - Previous Exp8 failure (Feb 21) remains unresolved
* **Lessons Learned**:
  1. Background Claude Code tasks may claim completion without actually completing all required work
  2. Task specification must be extremely explicit about deliverables and completion criteria
  3. Need to validate performance metrics immediately after Control phase before proceeding
  4. Should implement FM/DeepFM/Attention models as separate, testable modules before full experiment
  5. Task files should include explicit phase completion validation (e.g., "Do not proceed until Control Sharpe ≥ 0.48")
* **Next Steps Needed**:
  1. ✅ Investigated why Control performance degraded 58% from Exp7
  2. ✅ Fixed 3 bugs in models.py:
     - Line 2204: `return_returns == 'return'` → `return_returns == 'return_pred'`
     - Line 1705-1709: Added fm_only, deepfm, self_attention_1, self_attention_2 to fit() PyTorch check
     - Line 2626-2630: Added fm_only, deepfm, self_attention_1, self_attention_2 to predict() multiprocessing check
  3. ✅ Verified FMModel class exists (line 970), fm_only model creation uses FMModel (correct)
  4. ✅ Code ready for testing - all 3 bugs fixed
  5. ❌ Investigation test stuck: Control test (5 epochs) hung at Epoch 2/5 for 4.5 hours
  6. 🔴 Task-13 FAILED: Session stuck, root cause unknown. May be:
     - Training loop infinite iteration
     - MPS GPU compatibility issue
     - Model state not saving/loading properly
     - Data generator blocking
  7. Next: Need simpler test approach before full Exp8 retry:
     - Test with minimal epochs (1-2)
     - Add explicit logging at each step
     - Verify MPS GPU training works
     - Test data pipeline separately
