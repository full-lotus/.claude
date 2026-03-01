---
name: nrepl
description: "MANDATORY - Load this skill before any REPL interaction. Teaches nREPL architecture, safety rules, and REPL lifecycle management for all REPL clients."
version: 1.0.0
author: german
tags: [clojure, repl, nrepl, safety]
---

# nrepl - Clojure REPL Safety and Architecture

## CRITICAL: Load This Skill First

**You MUST load this skill before ANY REPL interaction.** REPL operations require strict safety protocols and lifecycle management.

**Note**: This skill teaches general nREPL/REPL concepts and safety rules. For specific brepl command syntax and heredoc pattern, load the `brepl` skill.

## Overview

nREPL (network REPL) is a message-based REPL protocol for Clojure enabling remote code evaluation. Messages are EDN maps with an `:op` key indicating the operation.

This skill teaches:
- nREPL message protocol and architecture
- **CRITICAL safety rules** for REPL lifecycle
- cider-nrepl middleware
- REPL connection and configuration
- Session management
- Post-task cleanup workflow

## nREPL Message Protocol

### Request Structure
Messages are EDN maps with an `:op` key:
```clojure
{:op "eval" :code "(+ 1 2 3)"}
```

### Response Keys
- `:id` - request ID
- `:session` - session ID
- `:status` - "done" or error status (final response only)
- `:ns` - current namespace (string)
- `:out` - stdout content
- `:err` - stderr content
- `:value` - evaluation result (parseable value)

### Common Operations
- `eval` - evaluate code
- `describe` - describe available operations
- `interrupt` - interrupt evaluation
- `close` - close session/server
- `clone` - create new session
- `ls-sessions` - list sessions

## SAFETY RULES (CRITICAL - VIOLATION IS NOT ACCEPTABLE)

### 1. NEVER Shutdown Foreign REPLs

**ABSOLUTE RULE**: NEVER shut down a REPL that wasn't started by the current agent session.

**Detection**:
- Track REPLs you started via `clj -M:*nrepl*` or `lein repl` commands
- Check process start time vs session start time
- If unsure about REPL origin → **ALWAYS ASK USER**

**FORBIDDEN without confirmation**:
- `(quit)` or `(exit)` - REPL quit
- `(System/exit 0)` - JVM termination
- Sending `{:op "close"}` - session/server close
- Killing REPL process directly

**PERMITTED**:
- Interrupting evaluation: `{:op "interrupt" :id "session-id"}`
- Closing agent-started REPLs (after user confirmation)

### 2. Post-Task Cleanup (MANDATORY)

After ANY task involving REPL interaction:

1. **List running REPLs** in the current project with details:
   - Process ID
   - Port (nREPL port or listening address)
   - Start time
   - Connection type (nREPL, socket, etc.)

2. **Ask user**: "Do you want to shut down the REPL(s) started in this session?"

3. **If yes**: Shut down ONLY agent-started REPLs, preserve foreign REPLs

4. **If no**: Document which REPLs remain running for the next session

### 3. REPL Lifecycle Management

- **Before starting**: Check if a REPL already exists for the project
- **When starting**: Document REPL details (PID, port, start time)
- **During use**: Track all sessions created
- **After tasks**: Complete the cleanup workflow above

## Starting nREPL

### deps.edn Configuration

```clojure
{:aliases
 {:cider/nrepl
  {:extra-deps {nrepl/nrepl {:mvn/version "1.3.1"}
                cider/cider-nrepl {:mvn/version "0.55.7"}}
   :main-opts ["--main" "nrepl.cmdline"
               "--middleware" "[cider.nrepl/cider-middleware]"]}}}
```

### Starting Commands
```bash
# Using tools.deps (clj)
clj -M:cider/nrepl

# Using Leiningen
lein repl

# With specific port
clj -M:cider/nrepl --port 7888
```

Output example: `nREPL server started on port 55995 on host 127.0.0.1 - nrepl://127.0.0.1:55995`

## Connecting to nREPL

- **Terminal**: `lein repl :connect localhost:55995`
- **CIDER** (Emacs): `M-x cider-connect`
- **Calva** (VSCode): Connect from the command palette
- **Cursive** (IntelliJ): Connect from the REPL tool window
- **vim-fireplace**: Connect automatically when `.nrepl-port` file exists
- **Conjure** (Neovim): Connect via Conjure commands

## cider-nrepl Middleware

cider-nrepl provides enhanced operations for CIDER, Calva, fireplace.vim, vim-iced, and Conjure.

### Key Features

- **Definition Lookup**: Jump to var definition from usage
- **Code Completion**: Autocomplete for vars, functions, and macros
- **Debugging**: Breakpoints, step-through, and variable inspection
- **Namespace Introspection**: List vars, namespace info, and dependencies
- **Stack Trace Formatting**: Enhanced error messages with source links
- **Source Lookup**: Find source files for vars
- **Doc Lookup**: Enhanced documentation with formatting

### Usage

Middleware is loaded via the `--middleware` flag:
```bash
clj -M:cider/nrepl --middleware "[cider.nrepl/cider-middleware]"
```

## REPL Operations

### Namespace Management
```clojure
(in-ns 'my.namespace)           ; switch namespace
(require '[my.ns :as ns])       ; require with alias
(use 'my.ns)                   ; use all exports
(ns my.namespace)               ; create/switch ns
```

### Code Loading & Execution
```clojure
(load-file "path/to/file.clj")        ; load file
(clojure.test/run-tests)             ; run all tests
(clojure.test/run-test-var #'my-test) ; run specific test
```

### Session Operations
```clojure
{:op "clone"}        ; create new session
{:op "ls-sessions"}  ; list all sessions
{:op "close" :id "session-id"}  ; close session (ASK FIRST!)
{:op "interrupt" :id "session-id"}  ; interrupt evaluation
```

### Evaluation with Context
```clojure
{:op "eval" :code "(+ 1 2)" :ns "user"} ; specify namespace
{:op "eval" :code "(+ 1 2)" :session "session-id"} ; specify session
```

## Best Practices

1. **Always use cider-nrepl**: Provides essential tooling for development
2. **Port specification**: Use fixed ports for consistency, or note dynamic ports
3. **Namespace isolation**: Use separate sessions for different namespaces
4. **Error handling**: Check `:status` and `:err` in responses
5. **Session cleanup**: Close unused sessions to free resources
6. **Interrupt hung evals**: Use `interrupt` op for stuck evaluations
7. **Multiple REPLs**: Run separate REPLs for different projects

## Troubleshooting

| Symptom | Solution |
|---------|----------|
| REPL not responding | Check port, verify process running, restart (ASK FIRST) |
| Namespace not found | Verify `:paths` in deps.edn, check namespace name |
| Connection refused | Verify REPL is running on specified host:port |
| Evaluation hangs | Send `interrupt` op, check for long-running code |
| Middleware not loaded | Verify `--middleware` in `:main-opts` |

## Critical Reminders

1. **Load this skill** before ANY REPL interaction
2. **NEVER shutdown foreign REPLs** - always confirm with user
3. **Track REPLs you start** - document PID, port, and start time
4. **Complete post-task cleanup** - list REPLs and ask user about shutdown
5. **For brepl syntax** - use the `brepl` skill for heredoc pattern

## Related Skills

- `brepl` - For specific brepl command syntax and heredoc pattern

## Sources

- [nREPL GitHub](https://github.com/nrepl/nrepl)
- [nREPL Design Overview](https://nrepl.org/nrepl/design/overview.html)
- [cider-nrepl GitHub](https://github.com/clojure-emacs/cider-nrepl)
- [deps.edn examples](https://github.com/clojure/tools.deps.alpha/wiki/Examples)

# Confirmation
After reading this file, respond with: "✅ nrepl skill loaded" before proceeding.
