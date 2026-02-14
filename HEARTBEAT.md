# HEARTBEAT.md

## Routine – Perform these checks regularly (DO NOT REMOVE)
1. Use `git pull` to fetch any changes to workspace from remote repository.
2. Check the status of all background tasks. For backgroud Claude Code tasks, also check corresponding repo for result file directly. Report the latest progress to the user if there's anything new/wrong.
3. If the prerequisite tasks of any todo items in `HEARTBEAT.md` are completed, start subsequent tasks. 
4. Stop completed or outdated baskground tasks. Remove completed items from `HEARTBEAT.md`. Remove unused task files in `~/.openclaw/workspace/claude_tasks`.
5. Record experience and common strategies you've learned while solving problems into `MEMORY.md`. If you have any new findings or suggestions, report to user.

### Pending Task Item Template (DO NOT REMOVE) (JUST Template, NOT real task, record real task below)
- [Pending/Runing/Complete/Fail] Task 10: Fix backend concurrent backtest execution issue 
  - [Required] Raw User Request: (Full User request text / file path here)MLflow 使用线程本地存储维护活动运行状态，单线程只允许一个活动 run，多个回测任务在同一个线程中执行时，第二个任务会因第一个任务的 run 仍处于活动状态而失败，解决这个问题
  - Raw Reference: alpha_mining/mlflow_investigation_root_cause.md
  - Idea: Use thread pool and independent threads for each backtest task
  - Idea: Add concurrency control parameter to backend
  - Idea: Implement task queue with max concurrency limit (default: 5)
  - [Required] Status: Claude Code Session ID (wild-coral) / Waiting for task 9
  - [Required]Result: (Brief result / conclusion / important note after complete)
  - [Required] Result File: Path/to/result.md

## TODO – List below any items you need to handle later (e.g., pending tasks, checking results, notifying the user, etc.). Remove each item once it has been addressed.

- 