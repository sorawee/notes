See also https://docs.racket-lang.org/guide/phases.html

## require for-syntax vs provide for-syntax 

```racket 
;; foo
#lang racket

(module a racket
  (provide (for-syntax a))
  (define-for-syntax a 42))

(require 'a)
(begin-for-syntax
  (println a))
```

=

```racket 
;; bar
#lang racket

(module a racket
  (provide a)
  (define a 42))

(require (for-syntax 'a))
(begin-for-syntax
  (println a))
```

Similarly:

``` racket
;; foo
#lang racket

(module b racket
  (provide b)
  (define b 42))

(module a racket
  (require (submod ".." b))
  (provide (for-syntax a))
  (define-for-syntax a #'b))

(require 'a)

(define-syntax (mac stx) a)
(mac)
```

=

``` racket
;; bar
#lang racket

(module b racket
  (provide b)
  (define b 42))

(module a racket
  (require (for-template (submod ".." b)))
  (provide a)
  (define a #'b))

(require (for-syntax 'a))

(define-syntax (mac stx) a)
(mac)
```

## variable-reference->module-base-phase

`variable-reference->module-base-phase` returns the phase where the module is instantiated.
Regular `require` would instantiate the required module at the same level as the current level.
`(require (for-syntax ...))` would instantiate the required module at one level higher than the current level. And so on.


```racket
;; foo
#lang racket

(module b racket
  (list 'b (variable-reference->module-base-phase (#%variable-reference))))

(module a racket
  (require (submod ".." b))
  (list 'a (variable-reference->module-base-phase (#%variable-reference))))

(require 'a)
```

outputs:

```
'(b 0)
'(a 0)
```

but 

``` racket
;; bar
#lang racket

(module b racket
  (list 'b (variable-reference->module-base-phase (#%variable-reference))))

(module a racket
  (require (for-template (submod ".." b)))
  (list 'a (variable-reference->module-base-phase (#%variable-reference))))

(require (for-syntax 'a))
```

outputs:

```
'(a 1)
'(b 0)
```

In both cases, `b` is instantiated at level 0. We can get foo to output `'(a 1)`
by wrapping it inside `begin-for-syntax` and use `variable-reference->phase` instead.

### Why does variable-reference->module-base-phase matter?


``` racket
#lang racket

(module my-mod racket
  (provide a)
  (define a 1))

(require (rename-in 'my-mod [a a-at-phase-0])
         (for-syntax (rename-in 'my-mod [a a-at-phase-1])))

(list 0 0 (free-identifier=? #'a-at-phase-0 #'a-at-phase-1 0 0))
(list 0 1 (free-identifier=? #'a-at-phase-0 #'a-at-phase-1 0 1))
(list 1 0 (free-identifier=? #'a-at-phase-0 #'a-at-phase-1 1 0))
(list 1 1 (free-identifier=? #'a-at-phase-0 #'a-at-phase-1 1 1))
```

outputs:

``` 
'(0 0 #f)
'(0 1 #t)
'(1 0 #f)
'(1 1 #f)
```

as expected.

But let's add a twist. What if the comparison is done at level 1?

``` racket
#lang racket

(module my-mod racket
  (provide a)
  (define a 1))

(module a racket
  (require (rename-in (submod ".." my-mod) [a a-at-phase-0])
           (for-syntax (rename-in (submod ".." my-mod) [a a-at-phase-1])))
  (list 0 0 (free-identifier=? #'a-at-phase-0 #'a-at-phase-1 0 0))
  (list 0 1 (free-identifier=? #'a-at-phase-0 #'a-at-phase-1 0 1))
  (list 1 0 (free-identifier=? #'a-at-phase-0 #'a-at-phase-1 1 0))
  (list 1 1 (free-identifier=? #'a-at-phase-0 #'a-at-phase-1 1 1))

  (list 1 2 (free-identifier=? #'a-at-phase-0 #'a-at-phase-1 1 2))
  (list 2 1 (free-identifier=? #'a-at-phase-0 #'a-at-phase-1 2 1))
  (list 2 2 (free-identifier=? #'a-at-phase-0 #'a-at-phase-1 2 2)))

(require (for-syntax 'a))
```

outputs:

```
'(0 0 #f)
'(0 1 #f)
'(1 0 #f)
'(1 1 #f)
'(1 2 #t)
'(2 1 #f)
'(2 2 #f)
'(0 0 #f)
'(0 1 #f)
'(1 0 #f)
'(1 1 #f)
'(1 2 #t)
'(2 1 #f)
'(2 2 #f)
```

Barring how the second output is (unfortunately) duplicated, the difference is that comparing at (0, 1) doesn't work anymore. Instead, we need to compare at (1, 2). In general, we should compare things with `variable-reference->module-base-phase` as the "base".

For macros, this problem is somewhat mitigated by the fact that most modules use regular `provide` (i.e., no `for-syntax` or other relative levels), so `free-identifier=?` which defaults levels to `syntax-local-phase-level` usually does the right thing:

``` racket
#lang racket

(module my-mod racket
  (provide kw)
  (define kw 1))

(module a racket
  (provide foo)
  (require (submod ".." my-mod))
  (define-syntax (foo stx)
    (println (syntax-local-phase-level))
    (syntax-case stx (kw)
      [(_ kw) #'#t]
      [_ #'#f])))

(require (for-syntax 'a 'my-mod))

(begin-for-syntax
  (println (foo kw)))
```

outputs:

```
1
#t
#t
```

(with duplicated `#t` output)

However, when modules `(provide (for-syntax ...))`, the problem starts to manifest:

``` racket
#lang racket

(module my-mod racket
  (provide (for-syntax kw))
  (define-for-syntax kw 1))

(module a racket
  (provide foo)
  (require (submod ".." my-mod))
  (define-syntax (foo stx)
    (println (syntax-local-phase-level))
    (syntax-case stx (kw)
      [(_ kw) #'#t]
      [_ #'#f])))

(require (for-syntax 'a)
         'my-mod)

(begin-for-syntax
  (println (foo kw)))
```

outputs:

```
1
#f
#f
```

because in the module `a`, `kw` in the syntax object `stx` is in level 1, but `kw` that 
it has a reference to is undefined in level 1 (it's defined in level 2).

We can fix this in two ways:

1. Use `for-template` so that `kw` is defined in level 1:

   ```racket
   #lang racket
   
   (module my-mod racket
     (provide (for-syntax kw))
     (define-for-syntax kw 1))
   
   (module a racket
     (provide foo)
     (require (for-template (submod ".." my-mod)))
     (define-syntax (foo stx)
       (println (syntax-local-phase-level))
       (syntax-case stx (kw)
         [(_ kw) #'#t]
         [_ #'#f])))
   
   (require (for-syntax 'a)
            'my-mod)
   
   (begin-for-syntax
     (println (foo kw)))
   ```
   
2. Use `syntax-case*` with custom `free-identifier=?`:

   ```racket
   #lang racket
   
   (module my-mod racket
     (provide (for-syntax kw))
     (define-for-syntax kw 1))
   
   (module a racket
     (provide foo)
     (require (submod ".." my-mod))
     (define-syntax (foo stx)
       (println (syntax-local-phase-level))
       (syntax-case* stx (kw)
           (λ (a b) (free-identifier=? a b 1 2))
         [(_ kw) #'#t]
         [_ #'#f])))
   
   (require (for-syntax 'a)
            'my-mod)
   
   (begin-for-syntax
     (println (foo kw)))
   ```


## namespace-base-phase

To complicate things further, there's `namespace-base-phase`:

``` racket
#lang racket

(module b racket
  (list 'namespace-base-phase 'b (namespace-base-phase))
  (list 'module-base-phase 'b (variable-reference->module-base-phase (#%variable-reference))))

(module a racket
  (require (for-template (submod ".." b)))
  (list 'namespace-base-phase 'a (namespace-base-phase))
  (list 'module-base-phase 'a (variable-reference->module-base-phase (#%variable-reference))))

(require (for-syntax 'a))
(list 'namespace-base-phase 'main (namespace-base-phase))
(list 'module-base-phase 'main (variable-reference->module-base-phase (#%variable-reference)))
```

outputs:

```
'(namespace-base-phase a 1)
'(module-base-phase a 1)
'(namespace-base-phase b 0)
'(module-base-phase b 0)
'(namespace-base-phase main 0)
'(module-base-phase main 0)
```

However, `namespace-base-phase` has `current-namespace` as a parameter. It's not relative to the module, so they could be different:

``` racket
;; main.rkt
#lang racket
(let ([old-compile (current-compile)])
  (current-compile
   (λ (stx imm?)
     (println (list 'compile-namespace-base-phase (namespace-base-phase)))
     (println (list 'compile-module-base-phase (variable-reference->module-base-phase (#%variable-reference))))
     (println (list 'source (syntax-source stx)))
     (old-compile stx imm?))))

(dynamic-require "a.rkt" #f)
```

``` racket
;; a.rkt
#lang racket

(list 'namespace-base-phase 'a (namespace-base-phase))
(list 'module-base-phase 'a (variable-reference->module-base-phase (#%variable-reference)))
(dynamic-require-for-syntax "b.rkt" #f)
```

``` racket
;; b.rkt
#lang racket

(list 'namespace-base-phase 'b (namespace-base-phase))
(list 'module-base-phase 'b (variable-reference->module-base-phase (#%variable-reference)))
```

outputs:

```
'(compile-namespace-base-phase 0)
'(compile-module-base-phase 0)
'(source #<path:/Users/sorawee/phase/a.rkt>)
'(namespace-base-phase a 0)
'(module-base-phase a 0)
'(compile-namespace-base-phase 1)
'(compile-module-base-phase 0)
'(source #<path:/Users/sorawee/phase/b.rkt>)
'(namespace-base-phase b 1)
'(module-base-phase b 1)
```

Note that by switching `dynamic-require-for-syntax` to `(require (for-syntax ...))`, the output changes to:

```
'(compile-namespace-base-phase 0)
'(compile-module-base-phase 0)
'(source #<path:/Users/sorawee/phase/a.rkt>)
'(compile-namespace-base-phase 0)
'(compile-module-base-phase 0)
'(source #<path:/Users/sorawee/phase/b.rkt>)
'(namespace-base-phase b 1)
'(module-base-phase b 1)
'(namespace-base-phase a 0)
'(module-base-phase a 0)
```

The difference is that `require` triggers compilation at compile-time, but `dynamic-require` triggers compilation
at runtime. It makes perfect sense for `dynamic-require`/`eval` to (potentially) change `namespace-base-phase`. 
I'm still not totally sure why `namespace-base-phase` ought not to change for `require`.

### Why does namespace-base-phase matter?

At least, in the context of customizing `current-compile`, we need to respect `namespace-base-phase` when comparing ids inside 
a given syntax object. Usually, most identifiers are inside the `module` form, where `namespace-base-phase` is reset back to 0
(and initially appears to start at level 0 --- but this is not true when the module form is eval-ed),
so it might appear that `namespace-base-phase` doesn't really matter.
However, `eval`, `dynamic-require`, and friends could give a syntax object with weird `namespace-base-phase` to us, so we really need
to handle them with care.

In short, use `namespace-base-phase` for an identifier that you are _given_, 
and use `variable-reference->module-base-phase` for an identifier that are apparent _in the module_.

## How to use syntax/parse with general phase levels?

Like this:

```racket
#lang racket/base

(module my-mod racket/base
  (define a 1)
  (provide a))

(module test racket/base
  (require syntax/parse
           (for-meta 2 (rename-in (submod ".." my-mod) [a a-known]))
           (for-meta 10 (rename-in (submod ".." my-mod) [a stx])))

  ;; +2 relative to module-base-phase
  (define-literal-set lits
    #:phase 2
    (a-known))

  (define base-phase (variable-reference->module-base-phase (#%variable-reference)))

  (define stx-phase (+ 10 base-phase))

  (println (list 'base-phase base-phase))
  (println (list 'eq-using-free-id
                 (free-identifier=? #'stx #'a-known stx-phase (+ 2 base-phase))))
  (println
   (list 'eq-using-syntax-parse
         (syntax-parse #'stx
           #:literal-sets ([lits #:phase stx-phase])
           [a-known #t]
           [_ #f]))))

(require (for-syntax 'test))
```
