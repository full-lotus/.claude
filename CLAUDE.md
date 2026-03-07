# Load these skills at the start of every session
## High-level LLM agent behavior rules
@~/.claude/skills/karpathy-guidelines/SKILL.md
IMPORTANT: UNLESS OTHERWISE INSTRUCTED, rely on /karpathy-guidelines to guide the agent

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

# Clojure REPL Evaluation
The command `clj-nrepl-eval` is installed on your path for evaluating Clojure code via nREPL.
## Discover nREPL servers
`clj-nrepl-eval --discover-ports`
## Evaluate code
`clj-nrepl-eval -p <port> "<clojure-code>"`
With timeout (milliseconds)
`clj-nrepl-eval -p <port> --timeout 5000 "<clojure-code>"`

The REPL session persists between evaluations - namespaces and state are maintained.
Always use `:reload` when requiring namespaces to pick up changes.
