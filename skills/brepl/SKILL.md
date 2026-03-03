---
name: brepl
description: "MANDATORY - Load this skill BEFORE using brepl in any way. Teaches the heredoc pattern for reliable code evaluation."
tags: [clojure, brepl, nrepl, parentheses, parens, brackets]
---

# brepl - Evaluating Clojure Code
## CRITICAL: Load This Skill First
**You MUST load this skill before using brepl.** Do NOT attempt to use brepl without loading this skill first, or you will use incorrect syntax.
**Always load this skill before using brepl. Always use the heredoc pattern for all Clojure code evaluation.**
## The Heredoc Pattern - Default Approach
**Always use heredoc for brepl evaluation.** This eliminates quoting issues, works for all cases, and provides a consistent, reliable pattern.

# nREPL server management
## CRITICAL: NEVER shut down a REPL that wasn't started by the current agent session.
## IMPORTANT: brepl requires a running nREPL server
Use this command to start a new server, if there isn't one running for the current project:
```bash
bb nrepl-server
```

# brepl usage
## Syntax (Stdin - Recommended)
This is the simplest heredoc syntax - stdin feeds directly to brepl.
```bash
brepl <<'EOF'
(your clojure code here)
EOF
```
**Important**: Use `<<'EOF'` (with quotes) not `<<EOF` to prevent shell variable expansion.

## Examples
**Multi-line expressions**:
```bash
brepl <<'EOF'
(require '[clojure.string :as str])
(str/join ", " ["a" "b" "c"])
EOF
```

**Code with quotes**:
```bash
brepl <<'EOF'
(println "String with 'single' and \"double\" quotes")
EOF
```

**Reloading and testing**:
```bash
brepl <<'EOF'
(require '[myapp.core] :reload)
(myapp.core/some-function "test" 123)
EOF
```

**Complex data structures**:
```bash
brepl <<'EOF'
(def config
  {:database {:host "localhost"
              :port 5432
              :name "mydb"}
   :api {:key "secret-key"
         :endpoint "https://api.example.com"}})
(println (:database config))
EOF
```

**Running tests**:
```bash
brepl <<'EOF'
(require '[clojure.test :refer [run-tests]])
(require '[myapp.core-test] :reload)
(run-tests 'myapp.core-test)
EOF
```

## Loading Files
To load an entire file into the REPL:
```bash
brepl -f src/myapp/core.clj
```
After loading, you can evaluate functions from that namespace.

## Fixing Unbalanced Brackets
Use `brepl balance` to fix unbalanced brackets in Clojure files using parmezan:
```bash
# Fix file in place (default)
brepl balance src/myapp/core.clj

# Preview fix to stdout
brepl balance src/myapp/core.clj --dry-run
```
**ALWAYS try to use brepl balance first, when you run into bracket issues**.

## Common Patterns
### Namespace reloading
```bash
brepl <<'EOF'
(require '[myapp.core] :reload-all)
EOF
```

### Documentation lookup
```bash
brepl <<'EOF'
(require '[clojure.repl :refer [doc source]])
(doc map)
(source filter)
EOF
```

### Error inspection
```bash
brepl <<'EOF'
*e
(require '[clojure.repl :refer [pst]])
(pst)
EOF
```

## Critical Rules
1. **Always use heredoc** - Use the heredoc pattern for all brepl evaluations
2. **Quote the delimiter** - Always use `<<'EOF'` not `<<EOF` to prevent shell expansion
3. **No escaping needed** - Inside heredoc, write Clojure code naturally
4. **Multi-step operations** - Combine multiple forms in one heredoc block
5. **Write correct Clojure** - Ensure proper bracket balancing and valid syntax

## Resources
brepl documentation: https://github.com/licht1stein/brepl
