# Research Tips

* When an experiment has been running for a long time without results: check the correctness of the code, run a small-scale experiment to estimate the time, or split it into multiple parallelizable experiments.
* When using PyTorch, utilize the MPS device to accelerate computations.
* Maintain a clean git repository: only commit essential content. Do not commit outdated versions, temporary test files, or intermediate log files.
* Complete tasks one by one. Add remaining tasks and items to be confirmed later to HEARTBEAT.md, clearly marking dependencies. Only when you are idle (or waiting for some tasks to finish), consider starting parallel tasks that have no dependencies and do not interfere with each other.