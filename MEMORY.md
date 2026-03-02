# Research Tips

* Code Review Finding (2026-02-17): Static code review of alpha_mining project found 3 critical issues in `random_signal` strategy (undefined variables in predict(), missing imports in backtest()), plus frontend display and information architecture issues. Key risks: strategy availability, task deletion stability, duplicate trade prevention.

* Backend Refactoring (2026-03-01): Major architectural refactoring in progress. Created layered architecture: domain/infrastructure/services/api/cli. 47 new Python files created across all layers. main.py reduced from 3264 lines to ~2 lines - successfully split monolithic API. New schema validation engine, plugin registry, qlib gateway, execution engine, and service layer. All changes made in last 30 minutes - session actively working.

* Parameter Validation Refactoring (2026-03-02): Successfully refactored parameter validation across alpha_mining project. Created schema_validator.py and runtime_contract.py to centralize validation logic. Eliminated 400+ lines of duplicate validation code across strategies and trading methods. Key improvements: (1) Replaced silent auto-clamping with explicit error reporting, (2) Enforced strict schema validation rejecting unknown fields, (3) Separated concerns: schema definition vs validation logic vs business logic, (4) Improved testability with dedicated test_parameter_schema_contract.py. Removed deprecated ma_short/ma_long params from rank_risk_control_v1.

* When an experiment has been running for a long time without results: check the correctness of the code, run a small-scale experiment to estimate the time, or split it into multiple parallelizable experiments.
* When using PyTorch, utilize the MPS device to accelerate computations.
* Maintain a clean git repository: only commit essential content. Do not commit outdated versions, temporary test files, or intermediate log files.
* Complete tasks one by one. Add remaining tasks and items to be confirmed later to HEARTBEAT.md, clearly marking dependencies. Only when you are idle (or waiting for some tasks to finish), consider starting parallel tasks that have no dependencies and do not interfere with each other.
