---
name: clojure-best-practices
description: Idiomatic Clojure coding patterns, naming conventions, data structures, sequence operations, threading macros, destructuring, control flow, functions, concurrency/state, error handling, protocols/records, macros, transducers, anti-patterns, performance tips, testing, and tooling.
version: 1.0.0
author: kombinacija
tags: [clojure, idiomatic, patterns, functional, concurrency]
---

# Idiomatic Clojure: Compressed Reference

## Core Philosophy
- **Simplicity**: One concern per function. No incidental complexity.
- **Immutability**: All collections immutable by default. Transformations return new values.
- **Data-oriented**: Maps/vectors/sets as primary abstraction. Separate data from operations.
- **Functional**: Pure functions, higher-order fns, composition over mutation.
- **REPL-driven**: Build incrementally, verify constantly.

## Naming Conventions
```clojure
!counter      ; atom (mutable ref)
save-user!    ; mutating function
valid-email?  ; predicate
->UserRecord  ; constructor
```

## Data Structures
```clojure
{:key "val"}           ; map (hash)
[:a :b :c]             ; vector (indexed)
#{:a :b}               ; set
'(1 2 3)               ; list (linked)

;; Transformation (never mutation)
(assoc m :k v)
(dissoc m :k)
(update m :k f)
(conj coll item)       ; appends to vector, prepends to list
(merge m1 m2)
```

## Sequence Operations
```clojure
(map f coll)
(filter pred coll)
(reduce f init coll)
(mapcat f coll)        ; flatMap
(keep f coll)          ; map + remove nils
(group-by f coll)      ; -> {key [items]}
(partition-by f coll)
(sort-by f coll)
(frequencies coll)     ; {item count}
(into {} pairs)        ; build map from pairs
```

## Threading Macros
```clojure
(-> x f1 f2 f3)        ; thread as first arg
(->> x f1 f2 f3)       ; thread as last arg
(as-> x $ (f1 $ 1) (f2 2 $))  ; named thread
(some-> x f1 f2)       ; short-circuit on nil
(cond-> x
  cond1 f1
  cond2 f2)            ; conditional threading
```

## Destructuring
```clojure
;; Sequential
(let [[a b & rest] [1 2 3 4]] ...)

;; Map
(let [{:keys [name age]} person] ...)
(let [{:keys [name] :as m} person] ...)
(let [{:strs ["name"]} string-map] ...)    ; string keys
(let [{:syms ['name]} symbol-map] ...)     ; symbol keys

;; Nested
(let [{:keys [address]
       {:keys [city]} :address} person] ...)

;; In fn args
(defn f [{:keys [a b]}] ...)
(defn f [[first & rest]] ...)
```

## Control Flow
```clojure
(if cond then else)
(when cond & body)        ; implicit do, returns nil on false
(cond c1 v1 c2 v2 :else v)
(case x :a 1 :b 2 default)
(condp = x :a 1 :b 2)
(if-let [v (maybe-nil)] use-v else)
(when-let [v (maybe-nil)] use-v)
(if-some [v (maybe-nil)] ...)   ; only nil triggers else, not false
```

## Functions
```clojure
(defn name [args] body)
(defn name [a & rest] body)        ; variadic
(defn name ([x] ...) ([x y] ...))  ; arity overloading
(fn [x] body)                      ; anonymous
#(+ % %2)                          ; reader fn (% = %1)
(partial f arg1)                   ; partial application
(comp f g h)                       ; right-to-left composition
(juxt f g h)                       ; apply multiple fns, return vec of results
(memoize f)                        ; cache results
```

## Concurrency / State
```clojure
;; Atom: uncoordinated, sync
(def !state (atom {}))
(swap! !state update :count inc)
(reset! !state new-val)
@!state                            ; deref

;; Ref: coordinated, sync (STM)
(def !r (ref 0))
(dosync (alter !r inc) (alter !r2 dec))

;; Agent: uncoordinated, async
(def !a (agent 0))
(send !a inc)

;; Var: per-thread dynamic binding
(def ^:dynamic *ctx* nil)
(binding [*ctx* "value"] ...)
```

## Error Handling
```clojure
;; Prefer specific exception types
(try
  (risky-fn)
  (catch java.io.FileNotFoundException e
    (throw (ex-info "File missing" {:file f} e)))
  (catch Exception e
    (throw (ex-info "Unexpected" {:cause (.getMessage e)} e))))

;; ex-info creates structured exceptions
(ex-info "msg" {:key "data"})
(ex-data e)   ; retrieve data map
```

## Protocols & Records
```clojure
(defprotocol Describable
  (describe [this]))

(defrecord Point [x y]
  Describable
  (describe [this] (str "(" x "," y ")")))

;; Prefer plain maps unless you need protocols/Java interop
;; deftype for Java interop, zero overhead
```

## Macros (use sparingly)
```clojure
;; Only when functions cannot capture of abstraction (e.g., syntax, laziness)
(defmacro unless [cond & body]
  `(when (not ~cond) ~@body))

;; Use gensym to avoid variable capture
(defmacro safe [& body]
  (let [result (gensym "result")]
    `(let [~result (do ~@body)] ~result)))
```

## Transducers (efficient pipelines)
```clojure
(def xf (comp (map inc) (filter even?) (take 10)))
(transduce xf conj [] (range 1000000))   ; no intermediate seqs
(into [] xf coll)
(sequence xf coll)                        ; lazy
```

## Common Anti-patterns to Avoid
```clojure
;; ❌ Atom as accumulator
(let [!r (atom 0)]
  (doseq [n nums] (swap! !r + n))
  @!r)

;; ✅ Use reduce
(reduce + nums)

;; ❌ Deep nesting
(reduce + (map #(* % %) (filter even? (map :val data))))

;; ✅ Thread
(->> data (map :val) (filter even?) (map #(* % %)) (reduce +))

;; ❌ OO-style record methods
(defn User->validate [this] ...)

;; ✅ Data-oriented
(defn validate-user [user] ...)

;; ❌ Macro for what a function can do
(defmacro bad-add [x] `(+ ~x 5))

;; ✅ Just a function
(defn add-five [x] (+ x 5))
```

## Performance Tips
- Prefer `mapv`/`filterv` for eager vectors
- Use transducers for large pipelines (no intermediate seqs)
- `subindexed` for large PState collections in Rama
- `^:unsynchronized-mutable` fields in deftypes for perf-critical mutation
- `(type-hint ^String s)` to avoid reflection overhead

## Testing
```clojure
(deftest my-test
  (is (= expected actual))
  (is (thrown? ExceptionType (bad-call)))
  (testing "sub-case"
    (is (= 1 1))))

;; Property-based
(require '[clojure.test.check.properties :as prop])
(prop/for-all [v (gen/vector gen/int)]
  (= (sort v) (sort (shuffle v))))
```

## Tooling
- **cljfmt**: formatting
- **clj-kondo**: static analysis (catch errors before runtime)
- **eastwood**: linting
- **kibit**: idiomatic suggestions
- **cloverage**: test coverage
