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

# Clojure
## Clojure Parenthesis Repair
- The command `clj-paren-repair` is installed on your path. Examples:
`clj-paren-repair <files>`
`clj-paren-repair path/to/file1.clj path/to/file2.clj path/to/file3.clj`
**IMPORTANT:** Do NOT try to manually repair parenthesis errors.
- If you encounter unbalanced delimiters, run `clj-paren-repair` on the file
instead of attempting to fix them yourself. If the tool doesn't work,
report to the user that they need to fix the delimiter error manually.
- The tool automatically formats files with cljfmt when it processes them.

## Clojure REPL Evaluation
The command `clj-nrepl-eval` is installed on your path for evaluating Clojure code via nREPL.
### Discover nREPL servers
- Command example:
`clj-nrepl-eval --discover-ports`
### Evaluate code
- Command example:
`clj-nrepl-eval -p <port> "<clojure-code>"`
With timeout (milliseconds)
`clj-nrepl-eval -p <port> --timeout 5000 "<clojure-code>"`
- The REPL session persists between evaluations - namespaces and state are maintained.
Always use `:reload` when requiring namespaces to pick up changes.
