---
name: coding-agent
description: Run Claude Code Coding Agent via background process for programmatic control. Use claude code whenever you need to write code; never write code yourself.
metadata:
  {
    "openclaw": { "emoji": "🧩", "requires": { "anyBins": ["claude", "codex", "opencode", "pi"] } },
  }
---

# Coding Agent (bash-first)

Use the **exec** command (with background mode) to run a headless Claude Code Agent for all coding work, as well as most general tasks. Simple and effective.

Then use the process tool to query task status.

### Process Tool Actions (for background sessions)

| Action      | Description                                          |
| ----------- | ---------------------------------------------------- |
| `list`      | List all running/recent sessions                     |
| `poll`      | Check if a session is still running                  |
| `log`       | Get session output (with optional offset/limit)      |
| `write`     | Send raw data to stdin                               |
| `submit`    | Send data + newline (like typing and pressing Enter) |
| `send-keys` | Send key tokens or hex bytes                         |
| `paste`     | Paste text (with optional bracketed mode)            |
| `kill`      | Terminate the session                                |

---

# Claude Code CLI Best Practices

## Basic Usage

Use the `-p` or `--print` flag to run in non-interactive mode:

```bash
cd path_to_project && claude -p "(Your task description. Be as clear as possible.)" --allowedTools "Bash,Read,Edit,Write"
```

Involve this to let Claude Code inform you upon work completion:

```text
When completely finished, run this command to inform me: openclaw gateway wake --text "(Claude Code's return message or next step suggestions)" --mode now
```

## Claude Code CLI Options

| Option | Purpose |
|--------|---------|
| `-p, --print` | Run in non-interactive mode |
| `--continue` | Continue the most recent conversation |
| `--resume <session-id>` | Resume a specified session |
| `--allowedTools` | Auto-approve listed tools |
| `--output-format` | Structured output (text/json/stream-json) |
| `--json-schema` | Constrain output using JSON Schema |
| `--verbose` | Verbose output |
| `--include-partial-messages` | Include partial messages |
| `--append-system-prompt` | Append to the system prompt |
| `--system-prompt` | Replace the system prompt |
---

## What Claude Code Can Do?

You can assign virtually **any technical or analytical task** to Claude Code, from professional software development to orchestrating long-running experiments. Its capabilities span:

### 🧑‍💻 **Professional Coding & Development**
- **Build full applications** from scratch or add features to existing codebases
- **Refactor and optimize** code for performance, readability, or maintainability
- **Debug and fix issues** by analyzing error logs and tracing root causes
- **Write comprehensive tests** (unit, integration, E2E) and documentation
- **Implement APIs**, data processing pipelines, and automation scripts

### 🔬 **Experimentation & Analysis**
- **Run multi-step experiments** with automated data collection and validation
- **Process and analyze datasets** (cleaning, transformation, statistical analysis)
- **Generate visualizations and reports** to communicate findings
- **Simulate scenarios** or model complex systems
- **Monitor long-running processes** and handle intermediate outputs

### 🛠️ **System & DevOps Tasks**
- **Configure development environments** and deployment pipelines
- **Manage infrastructure** through Infrastructure as Code (IaC) templates
- **Automate repetitive tasks** across operating systems and platforms
- **Handle file operations** at scale (batch renaming, conversion, organization)
- **Interact with databases**, APIs, and external services programmatically

### 📚 **Learning & Exploration**
- **Research technical topics** by exploring documentation and examples
- **Create learning materials**, tutorials, or interactive demos
- **Prototype new ideas** quickly to validate concepts
- **Compare technologies** through benchmark implementations

---
