# HEARTBEAT.md

## Routine – Perform these checks regularly (DO NOT REMOVE)
1. Use `git pull` to fetch any changes to workspace from remote repository.
2. Check status of all background tasks. For background Claude Code tasks, also check corresponding repository directly for result files. Report to latest progress to user if there's anything new/wrong.
3. If prerequisites of any todo items in `HEARTBEAT.md` are completed, start subsequent tasks.
4. Stop completed or outdated background tasks. Remove completed items from `HEARTBEAT.md`. Remove unused task files in `~/.openclaw/workspace/claude_tasks`.
5. Record experience and common strategies you've learned while solving problems into `MEMORY.md`. If you have any new findings or suggestions, report to user.

## TODO – List below any items you need to handle later (e.g., pending tasks, checking results, notifying user, etc.). Remove each item once it has been addressed.

- [Completed] Task Exp6: Multi-task Learning with Auxiliary Loss Functions
  - [Required] Raw User Request: Complete Experiment 6 (Multi-task Learning) from deeplearning_for_pair_wise_prediction.md
  - [Required] Status: COMPLETED - All 7/7 configs completed successfully
  - [Required] Prerequisite: Exp5 (Optimizer and Learning Rate) completed
  - [Required] Baseline Config: 4×64 MLP, ReLU, Adam, ReduceLROnPlateau scheduler, no dropout, no normalization, no residual, single_elim sorting, Standardize preprocessing
  - [Required] Experiment Design:
    - Control: Only Pairwise Ranking Loss
    - Exp A: Return Prediction (MSE loss on future N-day returns)
    - Exp B: Feature Reconstruction (masked input reconstruction)
    - Hyperparameters: λ ∈ {0.1, 0.5, 1.0} for each auxiliary loss
  - [Required] Deliverables: exp6_multitask_results.csv, exp6_multitask_report.md, task_exp6_completion_report.md
  - [Required] Archive location: task_history/260220/
  - [Required] Task File: claude_tasks/task_20260218_092000_exp6_multitask_loss.txt → task_history/260220/
  - [Required] Claude Session: fresh-crustacean (completed ~24 hours)
  - [Required] Completed at: 2026-02-20 05:42:45
  - [Required] Progress: 7/7 configs completed successfully
    - ✅ Control - Sharpe 0.3589 (8.55%)
    - ✅ Return Prediction (λ=0.1) - Sharpe 0.2914 (6.80%)
    - ✅ Return Prediction (λ=0.5) - Sharpe -0.0863 (-2.06%)
    - ✅ Return Prediction (λ=1.0) - Sharpe 0.1813 (4.54%)
    - ✅ Feature Reconstruction (λ=0.1) - Sharpe 0.0562 (1.29%)
    - ✅ Feature Reconstruction (λ=0.5) - Sharpe 0.2933 (6.80%)
    - ✅ Feature Reconstruction (λ=1.0) - Sharpe 0.2819 (6.44%)
  - [Required] Best Config: Control - 8.55% return, Sharpe 0.3589
  - [Required] Result File: alpha_mining/results/exp6_multitask_results_20260220_054245.csv
  - [Required] Report File: alpha_mining/results/exp6_multitask_report_20260220_054245.md
  - [Required] Completion Report: alpha_mining/results/task_exp6_completion_report_20260220_054245.md
  - [Required] Note: Task completed successfully from checkpoint after previous failure

- [Pending] Task Exp7: Data Augmentation
  - [Required] Raw User Request: Complete Experiment 7 (Data Augmentation) from deeplearning_for_pair_wise_prediction.md
  - [Required] Status: PENDING - Exp6 completed, ready to start
  - [Required] Experiment Design:
    - Control: No augmentation
    - Exp A: Cross-sectional Mixup
    - Exp B: Feature Noise (Gaussian noise N(0, ε))
  - [Required] Deliverables: exp7_augmentation_results.csv, exp7_augmentation_report.md, task_exp7_completion_report.md
  - [Required] Archive location: task_history/260220/

- [Pending] Task Exp8: Feature Interaction and Self-Attention
  - [Required] Raw User Request: Complete Experiment 8 (Feature Interaction and Self-Attention) from deeplearning_for_pair_wise_prediction.md
  - [Required] Status: PENDING - Depends on Exp7
  - [Required] Experiment Design:
    - Control: Standard MLP
    - Exp A: FM/DeepFM (Factorization Machine)
    - Exp B: Self-Attention (Transformer Encoder)
  - [Required] Deliverables: exp8_interaction_results.csv, exp8_interaction_report.md, task_exp8_completion_report.md
  - [Required] Archive location: task_history/260220/

- [Pending] Task Exp9: Final Experimental Report
  - [Required] Raw User Request: Generate comprehensive final report summarizing all experiments (Exp0-8)
  - [Required] Status: PENDING - Depends on Exp8
  - [Required] Deliverables: final_experiment_report.md, best_configuration_recommendation.md
  - [Required] Archive location: task_history/260220/

- [Pending] Task: Complete End-to-End Testing of Alpha Mining Platform
  - [Required] Raw User Request: Complete end-to-end testing of alpha_mining backend and frontend using Chrome DevTools MCP
  - [Required] Status: PENDING
  - [Required] Testing Scope:
    - Backend API Testing (health, factors, datasets, backtests, predictions, portfolios, simulation)
    - Frontend UI Testing (factor adjustment, backtest creation, view results, prediction, portfolio management, simulation, data market)
    - Chrome DevTools MCP simulation of user operations
  - [Required] Validation Criteria:
    - Must have end-to-end data results
    - No example/fake data
    - Real data visible in UI (no data = test failure)
    - Fix all found issues
    - Ensure functionality is end-to-end usable
  - [Required] Deliverables: testing/end_to_end_test_summary_*.md, backend_test_report_*.md, frontend_test_report_*.md, issues_found_*.md
  - [Required] Archive location: task_history/260220/
  - [Required] Note: Critical validation - real data results required for all tests
