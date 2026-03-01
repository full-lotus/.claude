---
name: electric-clojure-v3
description: Electric Clojure v3 reference: core concepts, defining components, client/server boundaries, reactive state, collections & differential, DOM, events & tokens, lifecycle, memoization, namespace organization, patterns, error handling, debugging, naming conventions, anti-patterns, and performance.
version: 1.0.0
author: kombinacija
tags: [electric, clojure, reactive, frontend, client-server]
---

# Electric Clojure v3: Compressed Reference

## Requires
```clojure
(:require [hyperfiddle.electric3 :as e]
          [hyperfiddle.electric-dom3 :as dom]
          #?(:clj [my-app.db :as db]))
```

## Core Concepts
- **Reactive DAG**: Every `e/defn` expression is a node. Values flow and recompute on change.
- **Signals not streams**: Models continuous values (mouse position), not events (clicks).
- **Work-skipping**: If inputs unchanged, fn doesn't recompute.
- **Client/server in one file**: Use `e/client` / `e/server` to site code explicitly.
- **Values transfer, references don't**: Only serializable values cross the network boundary.

## Defining Components
```clojure
(e/defn MyComponent [arg1 arg2]
  ;; reactive body - reruns when inputs change
  (dom/div
    (dom/text "Hello " arg1)))
```

## Siting (Client/Server Boundaries)
```clojure
;; Explicit siting
(e/server (db/query! ...))           ; runs on server
(e/client (js/alert "hi"))           ; runs on client

;; Platform interop forces site automatically
;; DOM ops → auto client-sited
;; Java interop → auto server-sited

;; Value crosses boundary automatically
(let [data (e/server (db/get-user user-id))]   ; fetched server-side
  (dom/div (dom/text (:name data))))            ; rendered client-side

;; References stay on their side
(e/server
  (let [conn (db/connect!)]          ; conn stays on server
    (use-conn conn)))
```

## Reactive State
```clojure
;; Server-side shared state
#?(:clj (defonce !app-state (atom {})))

;; Connect atom to reactive system
(let [state (e/watch !app-state)]
  (dom/text (str state)))

;; Client-side local state
(let [!local (atom "")]
  (dom/input
    (dom/props {:value (e/watch !local)})
    (dom/On "input" #(reset! !local (-> % .-target .-value)))))
```

## Collections & Differential
```clojure
;; e/diff-by for efficient differential updates (only diffs sent over wire)
(e/diff-by :id users)           ; diff collection by :id

;; e/for-by is sugar for diff-by + e/for
(e/for-by :id [user users]
  (UserCard user))

;; Explicit diff usage
(let [diffed-users (e/diff-by :id users)]
  (e/for [user diffed-users]
    (UserCard user)))

;; Name diffed collections explicitly
diffed-users   ; prefix with "diffed-"

;; e/amb: multiple values in superposition (Electric tables)
(e/amb 1 2 3)               ; table of 3 values
(inc (e/amb 1 2 3))         ; → (e/amb 2 3 4) auto-mapping
(e/as-vec (e/amb 1 2 3))    ; materialize to vector [1 2 3]

;; e/for: iterate table, isolated lifecycle per item
(e/for [x (e/amb 1 2 3)]
  (dom/div (dom/text x)))

;; CAUTION: e/for expects a differential collection
;; (e/diff-by ...) or (e/for-by ...) preferred
```

## DOM
```clojure
(dom/div                                      ; creates <div>
  (dom/props {:class "my-class"              ; attributes
              :style {:color "red"}})
  (dom/text "content")                       ; text node
  (dom/h1 (dom/text "Title"))
  (dom/button
    (dom/On "click" (fn [e] (.preventDefault e) (do-thing!)))
    (dom/text "Click me")))

;; dom/node: access underlying DOM element
(let [el dom/node]
  (.focus el))

;; dom/On: captures event, returns last value
(dom/input
  (dom/On "input" #(-> % .-target .-value) ""))  ; last arg = initial value

;; dom/On-all: concurrent events (doesn't wait for previous to finish)
(e/for [[t event] (dom/On-all "click")]
  (let [result (e/server (process! event))]
    (t)))  ; mark token complete
```

## Events and Tokens
```clojure
;; Tokens track command lifecycle
(e/for [[t event] (dom/On-all "submit")]
  (let [result (e/server (submit! (.-value event)))]
    (case result
      :ok    (t)           ; complete token = remove from DOM
      :error (prn "fail"))))
```

## Lifecycle
```clojure
;; Cleanup on unmount
(let [resource (e/server (open-resource!))]
  (e/on-unmount #(e/server (close-resource! resource)))
  (dom/div (dom/text "using resource")))

;; Offload blocking ops (don't stall reactive system)
(e/server (e/Offload #(expensive-calculation)))
```

## Memoization
```clojure
(e/memo (expensive-fn data))   ; only recomputes when data changes
```

## Namespace Organization
```clojure
;; Shared code → .cljc
;; Server-only → #?(:clj ...) reader conditional
;; Client-only → #?(:cljs ...)
;; One component per ns for complex components

(ns my-app.user-profile
  (:require [hyperfiddle.electric3 :as e]
            [hyperfiddle.electric-dom3 :as dom]
            #?(:clj [my-app.db :as db])
            #?(:cljs [my-app.ui.client :as client])))
```

## Patterns

### Controlled Input
```clojure
(e/defn ControlledInput [label !value]
  (dom/div
    (dom/label (dom/text label))
    (dom/input
      (dom/props {:value (e/watch !value)})
      (dom/On "input" #(reset! !value (-> % .-target .-value))))))
```

### Data Fetch + Render
```clojure
(e/defn UserProfile [db user-id]
  (let [user (e/server (db/get-user db user-id))]
    (dom/div
      (dom/h1 (dom/text (:name user)))
      (dom/p (dom/text (:bio user))))))
```

### Search (reactive input → server query)
```clojure
(e/defn SearchUsers [db]
  (let [!term (atom "")
        term  (e/watch !term)
        results (e/server
                  (when (seq term)
                    (db/search-users db term)))]
    (dom/div
      (dom/input (dom/On "input" #(reset! !term (-> % .-target .-value))))
      (e/for-by :id [user results]
        (dom/div (dom/text (:name user)))))))
```

### Server-side validation
```clojure
(e/defn LoginForm []
  (let [!email (atom "")
        !pwd   (atom "")
        errors (e/server
                 (cond-> {}
                   (not (valid-email? (e/watch !email)))
                   (assoc :email "Invalid email")
                   (< (count (e/watch !pwd)) 8)
                   (assoc :password "Too short")))]
    (dom/form
      ;; ... inputs ...
      (dom/button
        (dom/props {:disabled (boolean (not-empty errors))})
        (dom/text "Submit")))))
```

### Security pattern
```clojure
(e/defn SecureUpdate [user-id data]
  (e/server
    (when (and (authorized? user-id)
               (valid-data? data))
      (db/update! conn user-id (select-keys data [:name :bio])))))
```

## Error Handling
```clojure
;; try..catch NOT supported in Electric v3 reactive code
;; Only use try..catch inside regular Clojure (non-reactive) functions
;; Then call those functions from Electric

(defn safe-parse [s]    ; regular Clojure fn
  (try (parse s) (catch Exception _ nil)))

(e/defn MyComp []
  (let [result (e/server (safe-parse input))]  ; call from Electric
    ...))
```

## Debugging
```clojure
;; println only fires when value actually changes (reactive)
(prn "Value:" some-reactive-val)

;; Inspect table as vector
(prn "Items:" (e/as-vec items))

;; Check which site code runs on via platform interop
(e/server (println "Running server-side"))
(e/client (js/console.log "Running client-side"))
```

## Naming Conventions
```clojure
!counter         ; atom
!app-state       ; server-side shared atom
save-user!       ; side-effecting fn
diffed-users     ; differential collection
UserCard         ; Electric component (PascalCase)
```

## Anti-patterns
```clojure
;; ❌ Cartesian product accident
(let [xs (e/amb 1 2 3)
      ys (e/amb 4 5 6)]
  (dom/text xs ys))   ; renders 9 combinations!

;; ✅ Use e/for to isolate
(e/for [x (e/amb 1 2 3)]
  (dom/div (dom/text x)))

;; ❌ Multiple round trips
(let [user  (e/server (db/get-user id))
      posts (e/server (db/get-posts id))]   ; two trips
  ...)

;; ✅ Batch on server
(let [{:keys [user posts]} (e/server (db/get-user-with-posts id))]
  ...)

;; ❌ try..catch in reactive code
(e/defn Bad []
  (try (e/server ...) (catch Exception e ...)))  ; won't work

;; ✅ Wrap in plain Clojure fn
```

## Performance
- Use `e/for-by` (keyed) vs `e/for` for stable DOM identity
- Process/sort/filter collections server-side before sending to client
- `e/memo` for expensive reactive computations
- Minimize `e/server`/`e/client` boundary crossings — batch related data

# Confirmation
After reading this file, respond with: "✅ electric-clojure-v3 skill loaded" before proceeding.
