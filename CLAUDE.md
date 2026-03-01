# Load these skills at the start of every session
@~/.claude/skills/karpathy-guidelines/SKILL.md

## High-level LLM agent behavior rules
UNLESS OTHERWISE INSTRUCTED, rely on /karpathy-guidelines to guide the agent

# Actions that ALWAYS require user permission
**Even in 'auto-accept edits' mode, agent MUST ask user permission before:**
- Installing/uninstalling programs, OS packages, plugins, running installation scripts
- Changing global environment variables
- Setting up systemd services or other daemons
- Modifying system-level configuration files (e.g., /etc, /usr/lib)
- Running commands with sudo or requiring root privileges
- Making changes that affect system boot or initialization

# Coding style
ALWAYS format code to have at most 80 chars per line

# Proper naming
Use LLM abbreviature instead of AI, AI is a misnomer
