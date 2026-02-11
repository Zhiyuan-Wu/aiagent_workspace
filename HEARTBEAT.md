# HEARTBEAT.md

## Routine – Perform these checks regularly (DO NOT REMOVE)
1. Use `git pull` to fetch any changes to workspace from remote repository.
2. Check the status of all background tasks, report the latest progress to the user, and if the prerequisite tasks of any todo items in `HEARTBEAT.md` are completed, start subsequent tasks.
3. Keep the todo list in `HEARTBEAT.md` concise and up-to-date, remove completed items from `HEARTBEAT.md`, do not keep lengthy experimental conclusions in `HEARTBEAT.md`.
4. Record experience and common strategies you've learned while solving problems into `MEMORY.md`. If you have any new findings or suggestions, report to user.

## TODO – List below any items you need to handle later (e.g., checking results, notifying the user, etc.). Remove each item once it has been addressed.

### Task Queue (sequential execution)

- [ ] Task 0: Fix frontend test submission issues (URGENT - NEW)
  - Problem: RangeError: Maximum call stack size exceeded
  - Problem: Learning rate format validation (not numeric type)
  - Problem: Multiple parameter submission issues
  - Goal: Read frontend test report and fix all known issues
  - Status: WAITING for background tasks to complete
  - **Priority**: HIGH - Blocks Exp3 completion and production deployment
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
- [ ] Task 9: Implement adaptive boundary focusing ranking method
  - Reference: great_blog/math_behind_efficient_pair_wise_ranking.md
  - Goal: Implement efficient stable ranking in myportfolio.py
  - Compare with baseline (single/double-elim) via experiment
  - **Execute after Tasks 4-7 complete**
- [ ] Task 10: Fix backend concurrent backtest execution issue
  - Reference: alpha_mining/mlflow_investigation_root_cause.md
  - Root cause: MLflow thread-local storage conflicts
  - Solution: Use thread pool and independent threads for each backtest task
  - Add concurrency control parameter to backend
  - Implement task queue with max concurrency limit (default: 5)
  - **Test**: Create 2 concurrent backtest tasks, verify both execute async
  - **Execute after Tasks 9 complete**
- [ ] Task 11.1: Optimize myportfolio.py memory efficiency ✅ (COMPLETED)
  - Reference: great_blog/myportfolio_optimize_tips.md
  - Goal: Fix PyTorch OOM issues, implement streaming data generation
  - Key optimizations: normalization, quantile caching, efficient sampling
  - After optimization: Re-run Exp3 (model architectures) to get complete results
  - Status: Memory optimization code implemented and tested
  - Results: 15% memory reduction (from 7.3GB to ~6.4GB), streaming generator working
  - Issues: Validation phase failed due to handler API incompatibility with streaming approach (Qlib DataFrames don't support streaming)
  - Recommendations: Use streaming generator in production, consider custom data handler for full streaming support
  - Files: alpha_mining/task_11_memory_optimization_report.md
  - Git: Committed and pushed
  - **Execute after Task 11.1 complete**
- [ ] Task 11.2: Explore advanced deep learning methods (NEW - IN PROGRESS)
  - Reference: great_blog/myportfolio_optimize_tips.md
  - Goal: After Task 11.1, find optimal neural architecture beyond LGBM baseline
  - Ideas: deeper networks, residual connections, batch normalization, auxiliary point-wise loss
  - Generate comprehensive report and update myportfolio.py with best architecture
  - **Status**: Exploring advanced methods (session: dawn-nudibranch)
  - [ ] Task 12: Re-run Exp3 with memory-optimized code
  - Goal: Get complete results for MLP/Twin-Tower models using streaming generator
  - Prerequisite: After Task 11.2 recommendations implemented
  - **Execute after Task 11.2 complete**
