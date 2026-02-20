# HEARTBEAT.md

## Routine – Perform these checks regularly (DO NOT REMOVE)
1. Use `git pull` to fetch any changes to workspace from remote repository.
2. Check status of all background tasks. For background Claude Code tasks, also check corresponding repository directly for result files. Report to latest progress to user if there's anything new/wrong.
3. If prerequisites of any todo items in `HEARTBEAT.md` are completed, start subsequent tasks.
4. Stop completed or outdated background tasks. Remove completed items from `HEARTBEAT.md`. Remove unused task files in `~/.openclaw/workspace/claude_tasks`.
5. Record experience and common strategies you've learned while solving problems into `MEMORY.md`. If you have any new findings or suggestions, report to user.

## TODO – List below any items you need to handle later (e.g., pending tasks, checking results, notifying user, etc.). Remove each item once it has been addressed.

- [Archived] Task Exp6: Multi-task Learning with Auxiliary Loss Functions
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
  - [Required] Best Config: Control - 8.55% return, Sharpe 0.3589
  - [Required] Result File: alpha_mining/results/exp6_multitask_results_20260220_054245.csv
  - [Required] Report File: alpha_mining/results/exp6_multitask_report_20260220_054245.md
  - [Required] Note: Task completed successfully from checkpoint after previous failure

- [Running] Task Exp7: Data Augmentation
  - [Required] Raw User Request: Complete Experiment 7 (Data Augmentation) from deeplearning_for_pair_wise_prediction.md
  - [Required] Status: RUNNING - Started at 2026-02-20 08:07:45
  - [Required] Prerequisite: Exp6 completed
  - [Required] Experiment Design:
    - Control: No augmentation (baseline from Exp6)
    - Exp A1: Cross-sectional Mixup (α=0.2)
    - Exp A2: Cross-sectional Mixup (α=0.4)
    - Exp B1: Feature Noise (Gaussian noise N(0, 0.01))
    - Exp B2: Feature Noise (Gaussian noise N(0, 0.05))
    - Exp B3: Feature Noise (Gaussian noise N(0, 0.10))
  - [Required] Configuration: Best from Exp6 (Control + Return Prediction λ=0.5)
  - [Required] Deliverables: exp7_augmentation_results.csv, exp7_augmentation_report.md, task_exp7_completion_report.md
  - [Required] Archive location: task_history/260220/
  - [Required] Task File: claude_tasks/task_20260220_080500_exp7_augmentation.txt
  - [Required] Claude Session: Running in background (PID: 10671)
  - [Required] Estimated Time: ~4-6 hours
  - [Required] Progress: Experiment 1/6 (Control) - Handler built, training starting
  - [Required] Log: results/exp7_augmentation_log_20260220_080745.txt

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
