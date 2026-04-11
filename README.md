# ast

Structural annotations for Carp source forms. A macro-time library that
walks a quoted form and produces a dynamic map describing its shape
and walks it back to reconstruct a (possibly rewritten) form.

```clojure
; Fold `(if <literal-bool> a b)` to the taken branch.
(defndynamic fold-if [form]
  (let [a (AST.ann form)]
    (if (and (= (Map.get a 'type) 'if)
             (= (Map.get (Map.get a 'condition) 'type) 'boolean))
      (if (Map.get (Map.get a 'condition) 'value)
        (AST.unann (Map.get a 'then))
        (AST.unann (Map.get a 'else)))
      form)))

(fold-if '(if true 1 (expensive-thing)))  ; => 1
(fold-if '(if false (crash) "fallback"))  ; => "fallback"
(fold-if '(if x 1 2))                     ; => (if x 1 2), unchanged
```

## Installation

```clojure
(load "git@github.com:carpentry-org/ast@0.1.0")
```

## Usage

`AST.ann` takes a quoted form and returns a dynamic map describing it.
Every annotation has a `'value` slot holding the original form and a
`'type` tag naming the variant; each variant carries variant-specific
child keys.

```clojure
(load "ast.carp")

(AST.ann '(if true 1 2))
; => {'value '(if true 1 2) 'type 'if
;     'condition {'value true 'type 'boolean}
;     'then {'value 1 'type 'number 'width 'int}
;     'else {'value 2 'type 'number 'width 'int}}
```

`AST.unann` takes an annotation and rebuilds the source. The round-trip
is an identity:

```clojure
(= '(defn f [a b] (let [c (+ a b)] (if (> c 0) c 0)))
   (AST.unann
     (AST.ann '(defn f [a b] (let [c (+ a b)] (if (> c 0) c 0))))))
; => true
```

Because `unann` reads variant-specific child keys rather than the
stashed `'value` slot, you can mutate an annotation before reversing it
to rewrite source:

```clojure
; Swap the then and else branches of an `if`
(let [a (AST.ann '(if true 1 99))]
  (AST.unann
    (Map.put
      (Map.put a 'then (Map.get a 'else))
      'else (Map.get a 'then))))
; => (if true 99 1)
```

## Covered forms

Every Carp special form plus every literal kind has a variant tag:

| Category | Tags |
|---|---|
| Literals | `symbol` `string` `boolean` `character` `number` `unit` `array` |
| Bindings | `def` `defn` `let` `fn` |
| Types | `deftype` `register` `register-type` |
| Modules | `defmodule` `with` |
| Control flow | `do` `while` `if` `match` `match-ref` `break` |
| Value ops | `quote` `ref` `deref` `the` `set!` |
| Other | `application` |

`AST.ann` docstring enumerates the per-variant fields; use
`(print-doc AST.ann)` at the REPL.

Unknown forms trigger a `macro-error`.

## Testing

```
carp -x tests/ast.carp
```

<hr/>

Have fun!
