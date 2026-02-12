# HEARTBEAT.md

## Routine – Perform these checks regularly (DO NOT REMOVE)
1. Use `git pull` to fetch any changes to workspace from remote repository.
2. Check the status of all background tasks. For backgroud Claude Code tasks, also check corresponding repo for result file directly. Report the latest progress to the user, and 
3. If the prerequisite tasks of any todo items in `HEARTBEAT.md` are completed, start subsequent tasks. Stop completed baskground tasks.
4. Keep the todo list in `HEARTBEAT.md` concise and up-to-date, remove completed items from `HEARTBEAT.md`, do not keep lengthy experimental conclusions in `HEARTBEAT.md`.
5. Record experience and common strategies you've learned while solving problems into `MEMORY.md`. If you have any new findings or suggestions, report to user.

### Pending Task Item Template (DO NOT REMOVE) (JUST Template, NOT real task, record real task below)
- [⌛️/🔄/✅/⚠️] Task 10: Fix backend concurrent backtest execution issue 
  - Raw User Request: (Full User request here, do not )MLflow 使用线程本地存储维护活动运行状态，单线程只允许一个活动 run，多个回测任务在同一个线程中执行时，第二个任务会因第一个任务的 run 仍处于活动状态而失败，解决这个问题
  - Reference: alpha_mining/mlflow_investigation_root_cause.md
  - Idea: Use thread pool and independent threads for each backtest task
  - Idea: Add concurrency control parameter to backend
  - Idea: Implement task queue with max concurrency limit (default: 5)
  - Status: Waiting previous task finished / Waiting Task xx Finished / Runing (id: fast-bob), Check backgroud task progress regularly
  - Result: (Brief result / conclusion / important note after complete)

## TODO – List below any items you need to handle later (e.g., pending tasks, checking results, notifying the user, etc.). Remove each item once it has been addressed.

-

