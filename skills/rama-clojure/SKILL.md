---
name: rama-clojure
description: Rama Clojure distributed backend platform reference: modules, depots, PStates, topologies (stream/microbatch), dataflow language, query topologies, path API, aggregators, testing, and design principles.
version: 1.0.0
author: kombinacija
tags: [rama, clojure, distributed, backend, event-sourcing]
---

# Rama Clojure: Compressed Reference

## Overview
Rama = distributed backend platform. Replaces multiple databases + message queues + ETL pipelines.
Event sourcing model: append to **depots** → process in **topologies** → store in **PStates** → query via clients or **query topologies**.

## Namespace Imports
```clojure
(use 'com.rpl.rama)
(use 'com.rpl.rama.path)
(require '[com.rpl.rama.ops  :as ops])
(require '[com.rpl.rama.aggs :as aggs])
(require '[com.rpl.rama.test :as rtest])
```

## Variable Naming Conventions
```
*var     ; dataflow variable (value)
$$pstate ; PState reference
%op      ; anonymous operation
```

---

## Modules
```clojure
(defmodule MyModule [setup topologies]
  ;; declare everything here
  )

;; Optional module name override
(defmodule MyModule {:module-name "OtherName"} [setup topologies] ...)

(get-module-name MyModule)   ; → "com.mycompany/MyModule"
```

---

## Depots
Distributed, partitioned append-only logs. All data enters Rama through depots.

```clojure
;; Declaration
(declare-depot setup *depot :random)              ; random partition
(declare-depot setup *depot (hash-by :user-id))   ; hash partition - ensures ordering per key
(declare-depot setup *depot :disallow)            ; topology-only append
(declare-depot setup *depot :random {:global? true}) ; single partition

;; Tick depot (time-based, auto-fires)
(declare-tick-depot setup *ticks 60000)  ; every 60s

;; Custom partitioner
(defdepotpartitioner my-partitioner [data num-partitions]
  (mod (:some-field data) num-partitions))
```

**Key insight**: Co-locate depot partitioning with PState partitioning to avoid network hops.
```clojure
;; ✅ Colocated: depot and PState both partitioned by :user-id
(declare-depot setup *depot (hash-by :user-id))
;; then in topology: no extra |hash needed before writing $$user-pstate

;; ❌ Mismatched: requires extra |hash step (network hop)
(declare-depot setup *depot :random)
;; requires: (|hash *user-id) before writing to user-keyed PState
```

### Depot Client API
```clojure
(def depot (foreign-depot ipc module-name "*depot"))

;; Append (ack levels)
(foreign-append! depot data)                ; :ack (default) - waits for stream topology
(foreign-append! depot data :append-ack)    ; waits for persistence+replication only
(foreign-append! depot data nil)            ; fire-and-forget

;; Async
(foreign-append-async! depot data)          ; → CompletableFuture

;; Read partition
(foreign-object-info depot)                       ; {:num-partitions N}
(foreign-depot-partition-info depot 0)            ; {:start-offset X :end-offset Y}
(foreign-depot-read depot partition-idx start end)
```

### Depot Migrations
```clojure
(declare-depot setup *depot :random
  {:migration (depot-migration
                "migration-id-v1"
                (fn [record]
                  (if (= record 10)
                    DEPOT-TOMBSTONE     ; delete record
                    (str record))))})   ; transform record
```

---

## PStates
Durable, replicated, partitioned data structures (materialized views). Far more flexible than databases — arbitrary nested structure.

### Schema Declaration
```clojure
;; In a topology:
(declare-pstate s $$p Long)                   ; single value
(declare-pstate s $$p {String Long})          ; map<String,Long>
(declare-pstate s $$p {String {:subindex? true} Long})  ; subindexed values

;; Complex nested schemas
(declare-pstate mb $$profiles
  {Long                                        ; user-id key
   (fixed-keys-schema
     {:username String
      :display-name String
      :height-inches Long})})

;; Map with subindexed inner map (for large unbounded collections)
(declare-pstate mb $$outgoing-transfers
  {Long                                        ; user-id
   (map-schema String                          ; transfer-id
               (fixed-keys-schema {:to-user-id Long
                                   :amt Long
                                   :success? Boolean})
               {:subindex? true})})            ; subindex for efficient access

;; Schema helpers
(map-schema KeyClass ValueSchemaOrClass)
(fixed-keys-schema {:key1 Class1 :key2 Class2})
(vector-schema ElementClass)

;; PState options
(declare-pstate s $$p Long {:global? true})           ; single partition
(declare-pstate s $$p Long {:initial-value 0})        ; starting value
(declare-pstate s $$p Long {:private? true})          ; topology-only access
```

**Subindexing**: Use `{:subindex? true}` for collections with 1000s+ elements. Without it, reading one element reads whole collection. Subindexed structures are sorted — enables efficient range queries.

**Delete caution**: Deleting a parent of a subindexed structure orphans its elements on disk. Always delete nested subindexed structures explicitly first.

### PState Migrations
```clojure
(declare-pstate mb $$p
  {Long (migrated String "migration-v1" str)})  ; Long→String, migration id, transform fn

;; fixed-keys migration
(declare-pstate mb $$p
  {Long (migrated (fixed-keys-schema {:b String :c Long})
                  "update-keys"
                  (fn [m] (-> m (dissoc :a) (assoc :c 10) (update :b str)))
                  [(fixed-key-additions #{:c})
                   (fixed-key-removals #{:a})])})
```

### PState Client Queries (from outside module)
```clojure
(def pstate (foreign-pstate ipc module-name "$$p"))

;; Point query
(foreign-select-one (keypath :a :b) pstate)

;; Multi-value query
(foreign-select [ALL FIRST] pstate)           ; → seq of results

;; With explicit partition key
(foreign-select-one (keypath :k) pstate {:pkey "explicit-key"})

;; Range queries (on subindexed/sorted structures)
(foreign-select-one (sorted-map-range 3 11) pstate)           ; keys [3, 11)
(foreign-select-one (sorted-map-range-from :k 10) pstate)     ; up to 10 entries from :k
(foreign-select-one (sorted-map-range-to :k 10) pstate)       ; up to 10 entries before :k
(foreign-select-one (sorted-map-range-from-start 5) pstate)   ; first 5

;; Server-side aggregation in path (only count transferred, not whole map)
(foreign-select-one [(keypath :user :events) (sorted-map-range 0 100) (view count)] pstate)

;; Reactive proxy (auto-updates on change, fine-grained diffs)
(def proxy-val
  (foreign-proxy (keypath :a) pstate
    {:callback-fn (fn [newval diff oldval] (println "changed" diff))}))
@proxy-val   ; current cached value, no remote call

;; Always call .close when done
(.close proxy-val)

;; Async versions
(foreign-select-one-async path pstate)   ; → CompletableFuture
(foreign-proxy-async path pstate opts)
```

---

## Topologies

### Stream Topology
Low-latency (milliseconds), at-least-once, individual record processing.
```clojure
(let [s (stream-topology topologies "name")]
  (declare-pstate s $$p ...)
  (<<sources s
    (source> *depot :> *data)
    ;; ... dataflow ...
    ))
```

### Microbatch Topology
Higher throughput, hundreds-of-ms latency, **exactly-once semantics**. Prefer for analytics and when strict consistency required.
```clojure
(let [mb (microbatch-topology topologies "name")]
  (declare-pstate mb $$p ...)
  (<<sources mb
    (source> *depot :> %microbatch)
    (%microbatch :> *record)              ; explode batch → individual records
    ;; ... dataflow ...
    ))
```

In tests: `(rtest/wait-for-microbatch-processed-count ipc module-name "name" N)` to wait for N records.

### Source Options
```clojure
(source> *depot {:start-from :beginning} :> *data)
(source> *depot {:start-from :end} :> *data)       ; default
(source> *depot {:start-from (offset-ago 10 :records)} :> *data)
(source> *depot {:start-from (offset-after-timestamp-millis ts)} :> *data)

;; Stream only: retry mode
(source> *depot {:retry-mode :individual} :> *data)   ; default
(source> *depot {:retry-mode :all-after} :> *data)    ; retry all after failed
(source> *depot {:retry-mode :none} :> *data)         ; no retry
```

---

## Dataflow Language

### Basic Structure
"Call and emit" paradigm — operations emit zero, one, or many times to downstream code.
```clojure
(operation arg1 arg2 :> *output)          ; emit to default stream
(operation arg1 :stream-name> *output)    ; emit to named stream
(operation arg1 :> <anchor> *output)      ; create anchor for branching
```

### Variables
```clojure
*var    ; bind value
$$p     ; PState reference
%op     ; anonymous operation (closure)
```

### Conditionals
```clojure
(<<if condition
  (do-true-thing)
 (else>)
  (do-false-thing))

(<<cond
  (case> (> x 0))
  (do-positive)
  (case> (< x 0))
  (do-negative)
  (default>)
  (do-zero))

;; For inline expressions:
(ifexpr condition true-val false-val)
```

### Loops
```clojure
(loop<- [*i 0 :> *v]
  (:> *i)                           ; emit current value out of loop
  (<<if (< *i 5)
    (continue> (inc *i))))          ; next iteration

;; continue> can be called async (after partitioner) - loop continues
```

### Partitioners (Distributed Routing)
```clojure
(|hash *key)           ; route to task based on hash of *key
(|all)                 ; broadcast to all tasks
(|global)              ; route to global task (task 0)
(|origin)              ; route back to query origin (required at end of query topologies)
(|shuffle)             ; random task

;; PState-aware partitioning
(|hash$$ *key $$p)     ; partition same as PState $$p key
```

### PState Operations (in topology)
```clojure
;; Read
(local-select> (keypath *key) $$p :> *val)          ; local partition, sync
(local-select> [(keypath *k) ALL] $$p :> *item)     ; emits once per item
(select> (keypath *key) $$p :> *val)                ; partitions then reads

;; Write
(local-transform> [(keypath *key) (termval *val)] $$p)     ; set value
(local-transform> [(keypath *key) (term inc)] $$p)          ; apply fn
(local-transform> [(keypath *key) (term #(+ % *n))] $$p)   ; fn with closure
(local-transform> [(keypath *key) NONE>] $$p)               ; delete key
(local-transform> [(keypath *k1 *k2) (termval *v)] $$p)    ; nested key

;; Nil-safe reads
(local-select> [(keypath *key) (nil->val 0)] $$p :> *val)

(local-clear> $$p)     ; reset PState to initial value
```

### Aggregators
Declarative PState updates. Handle initialization automatically.
```clojure
;; In stream topology (not batch block)
(+compound $$p {*key (aggs/+sum *val)})
(+compound $$p {*key (aggs/+count)})
(+compound $$p {*key (aggs/+last *val)})
(+compound $$p {*key (aggs/+set-agg *val)})
(+compound $$p {*key (aggs/+set-remove-agg *val)})

;; Equivalent using local-transform (more verbose):
(local-transform> [(keypath *key) (nil->val 0) (term #(+ % *val))] $$p)

;; Top-N pattern
(aggs/+top-monotonic [N] $$top-list *tuple
  :+options {:id-fn first :sort-val-fn last})

;; Other aggregators
(aggs/+min *val :> *result)
(aggs/+max *val :> *result)
(aggs/+avg *val :> *result)    ; batch only, not into PState
(aggs/+map-agg *k *v :> *map)
(aggs/+vec-agg *val :> *vec)
(+NONE)                         ; remove element at position
```

### Batch Blocks (`<<batch`)
SQL-like: joins, aggregation with two-phase optimization. Required for `|global` aggregation to be efficient.

```clojure
(<<batch
  ;; Source 1
  (ops/explode [[:a 1] [:b 2]] :> [*k *v1])

  ;; Source 2 (joined on *k — inferred automatically)
  (gen>)
  (ops/explode [[:a 10] [:c 4]] :> [*k **v2])   ; **v2 = outer join (nullable)

  ;; Post-agg phase
  (println "Res:" *k *v1 **v2))

;; Subbatches for multi-stage
(defgenerator my-subbatch [microbatch]
  (batch<- [*key *count]
    (microbatch :> *record)
    (|hash (:id *record))
    (+compound $$counts {(:id *record) (aggs/+count :new-val> *count)})
    ;; *count available in post-agg due to :new-val>
    ))

;; In topology:
(<<batch
  (my-subbatch %microbatch :> *key *count)
  ;; use *key and *count
  )
```

**Two-phase aggregation**: Automatically enabled when ALL aggregators in batch block are **combiners** (like `+sum`, `+count`, `+max`). Dramatically reduces network traffic for global aggregations.

```clojure
;; ❌ Inefficient: sends all data to task 0
(|global)
(aggs/+sum *val :> *total)

;; ✅ Efficient: two-phase (local partial sum → combine at global)
(<<batch
  ...
  (|global)
  (aggs/+sum *val :> *total))
```

### Custom Operations
```clojure
;; Regular Clojure fn (return value = single emit)
(defn my-fn [x] (* x 2))
(my-fn *val :> *result)

;; deframaop: dataflow operation using reactive code
(deframaop multi-emit [*x]
  (:> (inc *x))
  (:> (dec *x)))

;; deframafn: must emit to :> exactly once (callable from Clojure too)
(deframafn compute [*x]
  (:> (* *x 2)))

;; Anonymous operations (capture lexical scope)
(<<ramafn %f [*a]
  (:> (+ *a *outer-val)))
(println "result:" (%f 1))

;; WARNING: Don't store anonymous ops (%f) in depots/PStates
;; Their class names change on module update

;; defoperation: Java-style with OutputCollector
(defoperation my-op [*x]
  (fn [output-collector]
    (.emit output-collector (inc *x))
    (.emitStream output-collector "other" (dec *x))))
```

### Branching & Unification
```clojure
(multi-emit *val :> <default> *v :other> <other> *v2)

;; Continue default branch
(println "default:" *v)

;; Switch to other branch
(hook> <other>)
(println "other:" *v2)

;; Merge branches (var must be defined on ALL branches to be in scope after)
(unify> <default> <other>)
(println "merged")
```

### Key Differences from Clojure
- No `.` Java interop — wrap in Clojure fns
- No `fn` — use `<<ramaop` / `<<ramafn`
- No `if`/`let` — use `<<if`, `loop<-`
- Constants embedded at compile time (not var indirection)
- All constants must be serializable

### Async
```clojure
;; Non-blocking async integration
(completable-future>
  (http-get-future *http-client *url)
  :> *response)
(process-response *response :> *result)

;; Yield if task thread over time limit (default 5ms)
(yield-if-overtime)
```

---

## Query Topologies
On-demand distributed queries. Define once, invoke like a function.

```clojure
;; Definition
(<<query-topology topologies "my-query" [*arg1 *arg2 :> *result]
  ;; dataflow — may partition across cluster
  (|hash *arg1)
  (local-select> (keypath *arg1) $$p :> *val)
  (compute *val *arg2 :> *result)
  (|origin))     ; REQUIRED: route result back to caller

;; Query MUST emit *result exactly once

;; Invocation from outside
(def query (foreign-query ipc module-name "my-query"))
(foreign-invoke-query query "arg1" "arg2")     ; blocking
(foreign-invoke-query-async query "arg1")      ; → CompletableFuture

;; Invocation from another topology
(invoke-query "my-query" *arg1 *arg2 :> *result)
```

**Leading partitioner optimization**: If first op is `|hash` on input var, query starts directly on the right task (saves a hop).

**Temporary PState**: `$$query-name$$` — per-invocation scratch space, cleared after query.

---

## Module Dependencies (Mirrors)
```clojure
(mirror-depot setup *other-depot "com.mycompany/OtherModule" "*depot")
(mirror-pstate setup $$p "com.mycompany/OtherModule" "$$p")
(mirror-query setup *q "com.mycompany/OtherModule" "query-name")
```

## Task Globals
```clojure
;; Large constants or external service clients available on all tasks
(declare-object setup *http-client (MyHttpClientTaskGlobal. nil))

;; Implement TaskGlobalObject for lifecycle
(deftype MyHttpClientTaskGlobal [...]
  TaskGlobalObject
  (prepareForTask [this task-id ctx] ...)  ; setup per task
  (close [this] ...))                      ; teardown
```

---

## Testing
```clojure
(deftest my-module-test
  (with-open [ipc (rtest/create-ipc)]
    (rtest/launch-module! ipc MyModule {:tasks 4 :threads 2})
    (let [module-name (get-module-name MyModule)
          depot  (foreign-depot  ipc module-name "*depot")
          pstate (foreign-pstate ipc module-name "$$p")
          query  (foreign-query  ipc module-name "my-query")]

      (foreign-append! depot {:user-id 1 :val "hello"})

      ;; For microbatch: must wait for async processing
      (rtest/wait-for-microbatch-processed-count ipc module-name "topology-name" 1)

      (is (= "hello"
             (foreign-select-one (keypath 1 :val) pstate)))

      (is (= 42 (foreign-invoke-query query 1))))))

;; Test pstate operations in isolation
(with-open [tp (rtest/create-test-pstate
                 {String (map-schema Long Long {:subindex? true})})]
  (rtest/test-pstate-transform [(keypath "a" 1) (termval 100)] tp)
  (println (rtest/test-pstate-select-one [(keypath "a" 1)] tp)))

;; Module updates
(rtest/update-module! ipc NewModuleVersion)

;; with-redefs works for constants
(with-redefs [my.ns/CONSTANT 3]
  (rtest/launch-module! ...))
```

---

## Ack Returns (Stream only)
Stream topologies can return values to client on `AckLevel.ACK`:
```clojure
;; In topology
(ack-return! "topology-name" *value)    ; return value to client

;; On client
(let [{"topology-name" result} (foreign-append! depot data)]
  (if result
    (println "Success, got:" result)
    (throw (ex-info "Failed" {}))))
```

---

## Complete Module Example: Bank Transfer
```clojure
(defrecord Transfer [transfer-id from-user-id to-user-id amt])
(defrecord Deposit  [user-id amt])

(defmodule BankTransferModule [setup topologies]
  (declare-depot setup *transfer-depot (hash-by :from-user-id))
  (declare-depot setup *deposit-depot  (hash-by :user-id))

  (let [mb (microbatch-topology topologies "banking")]
    (declare-pstate mb $$funds {Long Long})
    (declare-pstate mb $$outgoing-transfers
      {Long (map-schema String
                        (fixed-keys-schema {:to-user-id Long :amt Long :success? Boolean})
                        {:subindex? true})})
    (declare-pstate mb $$incoming-transfers
      {Long (map-schema String
                        (fixed-keys-schema {:from-user-id Long :amt Long :success? Boolean})
                        {:subindex? true})})

    (<<sources mb
      (source> *transfer-depot :> %microbatch)
      (%microbatch :> {:keys [*transfer-id *from-user-id *to-user-id *amt]})

      ;; Check funds on from-user partition (no race: events are serial per task)
      (local-select> [(keypath *from-user-id) (nil->val 0)] $$funds :> *funds)
      (>= *funds *amt :> *success?)

      ;; Deduct if successful
      (<<if *success?
        (<<ramafn %deduct [*curr] (:> (- *curr *amt)))
        (local-transform> [(keypath *from-user-id) (term %deduct)] $$funds))

      ;; Record outgoing transfer
      (local-transform> [(keypath *from-user-id *transfer-id)
                         (termval {:to-user-id *to-user-id :amt *amt :success? *success?})]
                        $$outgoing-transfers)

      ;; Move to to-user partition (may be on different machine)
      (|hash *to-user-id)

      ;; Credit funds if successful (exactly-once across partition boundary)
      (<<if *success?
        (+compound $$funds {*to-user-id (aggs/+sum *amt)}))

      ;; Record incoming transfer
      (local-transform> [(keypath *to-user-id *transfer-id)
                         (termval {:from-user-id *from-user-id :amt *amt :success? *success?})]
                        $$incoming-transfers)

      ;; Second source subscription in same topology
      (source> *deposit-depot :> %microbatch)
      (%microbatch :> {:keys [*user-id *amt]})
      (+compound $$funds {*user-id (aggs/+sum *amt)}))))
```

---

## Path API (com.rpl.rama.path)

### Common Navigators
```clojure
(keypath :a :b :c)           ; navigate to nested key
(keypath *k)                 ; dynamic key from variable
STAY                         ; stay at current position (select whole value)
ALL                          ; navigate to each element
FIRST / LAST                 ; first/last element
MAP-KEYS / MAP-VALS          ; all keys or values
(nthpath 0)                  ; index into list
(set-elem :x)                ; element in set

;; Filtering
(pred even?)                 ; continue only if predicate true
(pred= :value)               ; (pred #(= % :value))
(pred< 10)  (pred<= 10)
(pred> 10)  (pred>= 10)
(selected? path)             ; continue if path navigates to anything
(not-selected? path)

;; Nil handling
(nil->val 0)                 ; substitute 0 for nil
NIL->VECTOR                  ; nil → []
NIL->SET                     ; nil → #{}
NIL->LIST                    ; nil → ()

;; Terminal (write operations)
(termval *val)               ; set to value
(term inc)                   ; apply fn
(term #(+ % *n))             ; fn with closure
NONE>                        ; delete/remove element

;; Range (on subindexed sorted structures)
(sorted-map-range start end)                          ; [start, end)
(sorted-map-range start end {:inclusive-end? true})
(sorted-map-range-from start max-amt)
(sorted-map-range-to end max-amt)
(sorted-map-range-from-start max-amt)
(sorted-map-range-to-end max-amt)

;; Same for sets
(sorted-set-range ...)
(sorted-set-range-from ...)

;; Utilities
(view f)                     ; apply fn to navigated value (read-only)
(multi-path p1 p2 p3)        ; navigate multiple paths
(putval extra-arg)           ; add extra arg to transform fn
(if-path cond then else)
(filterer path)              ; subseq of elements matching path
```

### Specter select/transform (local Clojure, not in topologies)
```clojure
(select [ALL :name] users)                    ; → ["Alice" "Bob"]
(select-one [:a :b] m)                        ; → single value
(transform [:a :b] inc m)                     ; → updated map
(setval [:a :b] 42 m)                         ; → updated map
(multi-transform [:a (multi-path [:b (termval 1)] [:c (termval 2)])] m)
```

---

## Aggregators Quick Reference
```clojure
;; All usable in +compound or batch blocks
aggs/+count                    ; increment count
(aggs/+sum *val)               ; add values
(aggs/+min *val)               ; minimum
(aggs/+max *val)               ; maximum
(aggs/+last *val)              ; last seen
(aggs/+first *val)             ; first seen
(aggs/+set-agg *val)           ; build set
(aggs/+set-remove-agg *val)    ; remove from set
(aggs/+map-agg *k *v          ; build map
(aggs/+vec-agg *val)           ; build vector
(aggs/+avg *val)               ; average (batch only, no PState)
(aggs/+top-monotonic [N] $$p *obj :+options {:id-fn f :sort-val-fn f})
(+NONE)                        ; remove at position
(aggs/+limit [N] *v1 *v2 :+options {:sort *v1 :reverse? true})

;; Capture updated value (batch blocks only)
(aggs/+sum *val :new-val> *updated-total)
```

---

## ops Quick Reference
```clojure
(ops/explode *seq :> *elem)              ; emit once per element
(ops/explode-indexed *seq :> *i *elem)  ; with index
(ops/explode-map *map :> *k *v)         ; emit each kv pair
(ops/expand *tuple :> *a *b *c)         ; fixed-size expand
(range> *start *end :> *v)              ; like Clojure range, emits per element
(ops/sum nums)                           ; sum a collection
(vget $$p :> *val)                       ; get top-level PState value (class schema)

;; Async
(current-task-id)
(current-microbatch-id)
(yield-if-overtime)
```

---

## Design Principles
1. **Stream vs Microbatch**: Use microbatch unless you need sub-100ms latency. Microbatch is easier (exactly-once) and higher throughput.
2. **Depot partitioning**: Match depot partition key to PState access pattern to avoid network hops.
3. **Subindexing**: Always subindex maps/sets with 1000s+ elements.
4. **Batch blocks for global agg**: Wrap global aggregations in `<<batch` with combiner aggs for two-phase optimization.
5. **Serial task execution**: Only one event runs per task at a time — no race conditions for same-key operations.
6. **`|hash` before writes**: Always partition to the right task before writing to a PState.

# Confirmation
After reading this file, respond with: "✅ rama-clojure skill loaded" before proceeding.
