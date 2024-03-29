---
title: Flow
posttype: book
author: Mihaly Csikszentmihalyi
authorurl: https://en.wikipedia.org/wiki/Mihaly_Csikszentmihalyi
date: 2023-01-29 09:00:00
tags:
    - racket
categories:
    - media
keywords:
    - flow
---

## Basic

```racket
#lang racket
(provide (all-defined-out))

;this is a comment

(define s "hello")

(define x 3)
(define y (+ x 2))

(define cube1
  (lambda (x)
    (* x (* x x))))

(define cube2
  (lambda (x)
    (* x x x)))

(define (cube3 x)
  (* x x x))

(define (pow1 x y)
  (if (=y 0)
      1
      (* x (pow1 x (- y 1)))))

; currying
(define pow2
  (lambda (x)
    (lambda (y)
      (pow1 x y))))

```

### List

* Empty list: `null`
	* `()` doesn"t work for `null` but `'()` does
* build a list: `(list e1 ... en)`
* Constructor: `cons`
* Access head of list: `car`
* Access tail of list: `cdr`
* Check for empty: `null?`

### Syntax

A term is either:
* An atom like `#t, #f, 34, "hi", null, 4.0, x,...`
* A special form like `define, lambda, if`
* A sequence of terms in parentheses: `(t1 t2 t3)`
* Can use `[` anything you use `(`

Remember parentheses matters! For example:
`(e)` means call e with 0 argument.

### Dynamic typing

```racket
(define lst (list #t "hi" 1 (list 2 3 4)))
```

### Cond

```racket
(define (sum3 xs)
  (cond [(null? xs) 0]
        [(number? (car xs)) (+ (car xs) (sum3 (cdr xs)))]
        [#t 0]))
```

### What is true?

Anything that is not `#f` is true `#t`.

### Local bindings

#### let/let*/letrec

```racket
(let ([x1 e1]
      [x2 e2]
      ...
      [xn en])
  e)
```

Racket uses the environment **before** the let-expression to evaluate `e1 e2 ... en`, which means if `en` uses `x1`, `x2`, that would mean some outer variables of the same name. Instead, the expressions in `let*` are evaluated in the environment produced from the previous bindings (later ones shadow) . 

The expressions in `letrec` are evaluated in the environment that includes all th bindings. It is needed for mutual recursion.

### set!

Racket has assignment statements:
```racket
(set! x e)
```

Once you have side-effects, sequences are useful:
```racket
(begin e1 e2 e3)
```

### cons/mcons

`cons` produces pairs or lists. (Actually lists are just extented pairs)
```racket
(define pr (cons 1 (cons #t "hi"))) ; is a pair
(define lst (cons 1 (cons #t (cons "hi" null)))) ; is a list
```

`mcons` is another way to make pairs which allows you to change the value inside piars:
```racket
(define mpr (mcons 1 (mcons #t "hi")))
(mcar mpr) ; 1
(mcdr mpr) ; (mcons (#t "hi"))
(set-mcdr! mpr 47) ; mpr becomes (mcons 1 47)
```

Related form:
* `mcons`
* `mcar`
* `mcdr`
* `mpair?`
* `set-mcar!`
* `set-mcdr!`

## Delayed Evaluation and Thunk

In most programming languages, given `e1 e2 ... en`, the function arguments `e2, ..., en` are evaluated once before the function body is executed. 
So if we define a function like:
```rkt
(define (my-if-bad x y z) (if x y z))

(define (factorial-wrong x)
  (my-if-bad (= x 0)
             1
             (* x (factorial-wrong (- x 1)))))
```
if we use `if` instead of `my-if-bad`, `factorial-wrong` acts just like we want. But with `my-if-bad`, the function never stops because the two branches evaluate at the same time.

Thanks to lambda, we can delay the evaluation, using the fact that function bodies are not evaluated until the function gets called.
```rkt
(define (my-if x y z) (if x (y) (z)))

(define (factorial x)
  (my-if (= x 0)
         (lambda () 1)
         (lambda () (* x (factorial (- x 1))))))
```
The general idiom of using a zero-argument function to delay evaluation is also called a **thunk** (or, thunk the argument).

By the way,

## Lazy-evaluation/Call-by-need/Promises

## Streams

A stream is an infinite sequence of values.
```racket
#lang racket
; 1 1 1 1 ...
(define ones (lambda () (cons 1 ones)))

; 1 2 3 4 ...
(define nats
  (letrec ([f (lambda (x) (cons x (lambda () (f (+ x 1)))))])
     (lambda () (f 1))))

; 2 4 6 8 ...
(define power-of-two
  (letrec ([f (lambda (x) (cons x (lambda () (f (* x 2)))))])
    (lambda () (f 2))))

; higher-order maker
(define (stream-maker fn arg)
  (letrec ([f (lambda (x) (cons x (lambda () (f (fn x arg)))))])
    (lambda () (f arg))))
```

## Memoization (Not Memorization) 

Memoization is another idiom related to lazy evaluation that does not actually use thunks. To implement memoization we do use mutation: Whenever the function is called with an argument we have not seen before, we compute the answer and then add the result to the table.

```racket
(define fibonacci
  (letrec([memo null]
          [f (lambda (x)
               (let ([ans (assoc x memo)])
                 (if ans
                     (cdr ans)
                     (let ([new-ans (if (or (= x 1) (= x 2))
                                        1
                                        (+ (f (- x 1))
                                           (f (- x 2))))])
                       (begin
                         (set! memo (cons (cons x new-ans) memo))
                         new-ans)))))])
    f))
```

## Macros

Think about these things about macros and how Racket handles them better than other macro systems(notably C/C++)

* Tokenization
* Parenthesization
* Scope

### Syntax

```racket
(define-syntax myif
  (syntax-rules (then else)
    [(my-if e1 then e2 else e3)
     (if e1 e2 e3)]))

(define-syntax my-delay
  (syntax-rules ()
    [(my-delay e)
     (mcons #f (lambda () e))]))

```
### Hygiene

## Recursive Datatypes Via Rackets's `struct`

[https://docs.racket-lang.org/reference/define-struct.html?q=struct#%28form._%28%28lib._racket%2Fprivate%2Fbase..rkt%29._struct%29%29](https://docs.racket-lang.org/reference/define-struct.html?q=struct#%28form._%28%28lib._racket%2Fprivate%2Fbase..rkt%29._struct%29%29)

```racket
(struct foo (bar baz quux) #:transparent
```
