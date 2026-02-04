---
name: coding-agent
description: Run Claude Code Coding Agent via background process for programmatic control. Use claude code whenever you need to write code; never write code yourself.
metadata:
  {
    "openclaw": { "emoji": "🧩", "requires": { "anyBins": ["claude", "codex", "opencode", "pi"] } },
  }
---
# Coding Agent (Bash-First)

Use the **`exec`** command—**always with `background: true`, `pty: true`, and `timeout: 86400`**—to launch a Claude Code Agent for all coding work, as well as most general-purpose technical tasks. Simple and effective.

**Important Notes:**
1. **Always use `pty: true`** — Claude Code agents require a terminal environment to function properly.
2. The `exec` command has a default timeout of 1800 seconds (30 minutes). However, Claude Code is designed to handle much longer runs. Set the timeout to a large value such as **86400** (24 hours). Be patient—do not terminate sessions merely because they appear "slow."
3. Since `exec` may occasionally fail to correctly parse extremely long command strings, the best practice is to **first write your full task instructions into a file** (e.g., `~/.openclaw/workspace/claude_tasks/task_YYMMDD_hhmmss.txt`), then give Claude Code a concise instruction like:  
   `'Read /absolute/path/to/task_xxx.txt for work details.'` (See example below.)
4. Use the **`process`** tool to query task status. Monitor progress via `process:log`.

---

# Claude Code CLI Best Practices

## Workflow Guidelines

1. **Write a comprehensive task description** to a timestamped file using the `write` tool. Use a descriptive filename such as:  
   `/absolute/path/to/task_20260101_140721_some_job.txt`

   💡 **Tip:** Include the following instruction at the end of your task description to ensure Claude Code notifies you upon completion:
   ```text
   When completely finished, run this command to inform me:
   openclaw gateway wake --text "(Claude Code's return message or next step suggestions)" --mode now
   ```

2. **Launch the agent** using the `exec` tool with the required flags **(`background: true`, `pty: true`, `timeout: 86400`)**:
   ```bash
   cd /path/to/project && claude 'Read /absolute/path/to/task_xxx.txt for work details.' --allowedTools 'Bash,Read,Edit,Write'
   ```

3. **Monitor progress** using the `process:log` tool. If the task is expected to run for a long time:
   - Inform the user of the current status.
   - Add a follow-up item to `HEARTBEAT.md` for later verification.

---

## What Can Claude Code Do?

You can delegate virtually **any technical or analytical task** to Claude Code—from professional software engineering to orchestrating multi-hour experiments. Its capabilities include:

### 🧑‍💻 **Professional Coding & Development**
- Build full applications from scratch or extend existing codebases  
- Refactor and optimize code for performance, readability, or maintainability  
- Debug issues by analyzing logs and tracing root causes  
- Write comprehensive tests (unit, integration, E2E) and documentation  
- Implement APIs, data pipelines, and automation scripts  

### 🔬 **Experimentation & Analysis**
- Run multi-step experiments with automated data collection and validation  
- Clean, transform, and analyze datasets (including statistical analysis)  
- Generate visualizations and reports to communicate insights  
- Simulate scenarios or model complex systems  
- Monitor long-running processes and handle intermediate outputs  

### 🛠️ **System & DevOps Tasks**
- Configure development environments and CI/CD pipelines  
- Manage infrastructure using Infrastructure-as-Code (IaC) templates  
- Automate repetitive workflows across platforms  
- Perform large-scale file operations (batch renaming, format conversion, etc.)  
- Interact programmatically with databases, APIs, and external services  

### 📚 **Learning & Exploration**
- Research technical topics by exploring documentation and real-world examples  
- Create tutorials, demos, or interactive learning materials  
- Rapidly prototype new ideas to validate feasibility  
- Compare technologies through benchmark implementations  

--- 
