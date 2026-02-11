# HEARTBEAT.md

## Routine – Perform these checks regularly (DO NOT REMOVE)
1. Use `git pull` to fetch any changes to workspace from remote repository.
2. Check the status of all background tasks, report the latest progress to the user, and if the prerequisite tasks of any todo items in `HEARTBEAT.md` are completed, start subsequent tasks. Stop completed baskground tasks.
3. Keep the todo list in `HEARTBEAT.md` concise and up-to-date, remove completed items from `HEARTBEAT.md`, do not keep lengthy experimental conclusions in `HEARTBEAT.md`.
4. Record experience and common strategies you've learned while solving problems into `MEMORY.md`. If you have any new findings or suggestions, report to user.

## TODO – List below any items you need to handle later (e.g., checking results, notifying the user, etc.). Remove each item once it has been addressed.

### Active Background Task Monitoring
- Check Task 12 (fast-summit) progress regularly
- When Task 12 completes: Start Task 13 (first part: browser testing yourself, then delegate to Claude Code)

### Task Queue (sequential execution)

- [x] Task 1: Push myportfolio.py changes to remote repository ✅
- [x] Task 2: Add CLI parameters to myportfolio.py for:
  - Training dataset size (pairs per day)
  - Quantile filter threshold
  - Different models (LGBM, MLP, Twin-Tower)
  - Different tournament strategies (random, single-elim, double-elim, hybrid)
  - Number of tournament runs ✅
- [x] Task 3: Experiment on training dataset size impact (exp1 report) ✅
  - Results: k=10 best (Sharpe 0.8188, Return 17.95%)
  - Files in: exp1_results/
- [x] Task 4: Experiment on quantile filter threshold impact (exp2 report) ✅
  - Results: q=0.4 best (Sharpe 0.8392, Return 18.32%)
  - Files in: exp2_results/
- [x] Task 5: Experiment on different model architectures (exp3 report) ✅
  - Results: LGBM best (Sharpe 0.8807, Return 19.04%)
  - Note: MLP/Twin-Tower failed due to memory issues
  - Files in: exp3_results/
- [x] Task 6: Experiment on different tournament mechanisms (exp4 report) ✅
  - Results: Double-Elim best (Sharpe 1.018, Return 21.9%)
  - Files in: exp4_results/
- [x] Task 7: Experiment on tournament run count impact (exp5 report) ✅
  - Results: n=15 best (Sharpe 1.014, best stability)
  - Files in: exp5_results/
- [x] Task 8: Read all experiment reports and create final parameter analysis report, sync to Notion
  - Report created: alpha_mining/final_parameter_analysis_report.md ✅
  - Notion sync: Requires Notion skill/API key
- [x] Task 9: Implement adaptive boundary focusing ranking method ✅
  - Reference: great_blog/math_behind_efficient_pair_wise_ranking.md
  - Goal: Implement efficient stable ranking in myportfolio.py
  - Compare with baseline (single/double-elim) via experiment
  - **Results**: Adaptive method best (Sharpe 0.9021, Return 19.82%)
  - Files in: exp6_results/
- [x] Task 10: Fix backend concurrent backtest execution issue ✅
  - Reference: alpha_mining/mlflow_investigation_root_cause.md
  - Root cause: MLflow thread-local storage conflicts
  - Solution: Use thread pool and independent threads for each backtest task
  - Add concurrency control parameter to backend
  - Implement task queue with max concurrency limit (default: 5)
  - **Results**: Concurrent executor implemented, tests passed
- [x] Task 11.1: Optimize myportfolio.py memory efficiency ✅
  - Reference: great_blog/myportfolio_optimize_tips.md
  - Goal: Fix PyTorch OOM issues, implement streaming data generation, applied to myportfolio.py
  - Key optimizations: normalization, quantile caching, efficient sampling
  - **Results**: Training memory ~7.3GB→<500MB (93% reduction), validation memory ~1.6GB→<100MB (94% reduction)
- [x] Task 11.2: Explore advanced deep learning methods ✅
  - Reference: great_blog/myportfolio_optimize_tips.md
  - Goal: After Task 11.1, find optimal neural architecture beyond LGBM baseline, Do examperiments and give a full report.
  - Ideas: deeper networks, residual connections, batch normalization, auxiliary point-wise loss
  - **Results**: 6 architectures implemented and validated. Converged properly (loss 0.6873 → 0.6793), Peak memory 13.7GB. Ready for evaluation.
  - Task file: task_11_2_dl_exploration.txt
- [🔄] Task 12: Re-run Exp3 with memory-optimized code
  - Goal: Get complete results for MLP/Twin-Tower models using streaming generator
  - Prerequisite: After Task 11.1 applied, and After Task 11.2 recommendations implemented
  - Task file: task_12_re_exp3.txt
  - **Status**: Claude Code session started (fast-summit)
- [ ] Task 13: Fix frontend test submission issues
  - Problem: RangeError: Maximum call stack size exceeded
  - Problem: Learning rate format validation (not numeric type)
  - Problem: Multiple parameter submission issues
  - Goal: Use `browser` tool to test website and write down a frontend test report. Let Claude Code Read frontend test report and fix all known issues
  - Task file: task_13_frontend_fix.txt
  - **Execute after Task 12 complete**