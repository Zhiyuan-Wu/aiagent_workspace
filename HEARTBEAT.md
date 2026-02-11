# HEARTBEAT.md

## Routine – Perform these checks regularly (DO NOT REMOVE)
1. Use `git pull` to fetch any changes to workspace from remote repository.
2. Check the status of all background tasks, report the latest progress to the user, and if the prerequisite tasks of any todo items in `HEARTBEAT.md` are completed, start the subsequent tasks.
3. Keep the todo list in `HEARTBEAT.md` concise and up-to-date, remove completed items from HEARTBEAT.md, do not keep lengthy experimental conclusions in HEARTBEAT.md.
4. Record experience and common strategies you've learned while solving problems into `MEMORY.md`. If you have any new findings or suggestions, report to user.

## TODO – List below any items you need to handle later (e.g., checking results, notifying the user, etc.). Remove each item once it has been addressed.

- [ ] Monitor Claude Code background task (session: good-lobster) for myportfolio.py rewrite completion
  - Task: Rewrite myportfolio.py to correctly implement pair-wise ranking
  - Check progress every few minutes using `process:log`
  - When completed, verify results and push to remote repository
  - Expected notification: `openclaw gateway wake --text "..."`
