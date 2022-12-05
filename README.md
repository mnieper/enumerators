# Enumerators

An *enumerator* is a (higher-order) procedure representing a finite sequence.  Each element of the sequence is represented by multiple values.  An enumerator procedure takes two arguments, a procedure `combine` and a value `nil`.  Invoking an enumerator applies `combine` to `nil` and the multiple values representing the first element.  Then, `combine` is applied to the resulting value and the multiple values representing the second element and so on until the last element has been processed.  The result of the last application of `combine` (or `nil` if there was none) is returned.

## Examples for enumerators

```scheme
(define list-enumerator
  (lambda (ls)
    (assert (list? ls))
    (lambda (proc nil)
      (let f ((ls ls) (acc nil))
        (if (null? ls)
            acc
            (f (cdr ls) (proc acc (car ls))))))))

(define generator-enumerator
  (lambda (gen)
    (assert (procedure? gen))
    (lambda (proc nil)
      (let f ((acc nil))
        (define el (gen))
        (if (eof-object? el)
            acc
            (f (proc acc el)))))))

(define iota-enumerator
  (case-lambda
    ((count) (iota-enumerator count 0))
    ((count start) (iota-enumerator count start 1))
    ((count start step)
     (assert (and (integer? count) (exact? count) (not (negative? count)))
     (lambda (proc nil)
       (let f ((count count) (start start) (acc nil))
         (if (zero? count)
             acc
             (f (- count 1) (+ start step) (proc acc start))))))))
```

## Utility procedures

```scheme
(define enumerator-for-each
  (lambda (proc enum)
    (assert (procedure? proc))
    (assert (procedure? enum))
    (enum
     (lambda (acc . args)
       (apply proc args)
       #f)
     #f)))

(define enumerator-fold
  (lambda (proc nil enum)
    (assert (procedure? proc))
    (assert (procedure? enum))
    (enum proc nil)))
```

## Utility syntax

The syntax of `enumerator-for` can possibly be improved.

```scheme
(define-syntax enumerator-for
  (syntax-rules ()
    ((enumerator-for ((x init) ...)
         ((formals enum-expr) body)
       expr)
     (call-with-values
         (enum-expr
          (proc
           (lambda (thunk . formals)
             (call-with-values thunk
               (lambda (x ...)
                 (let-values (((x ...) body))
                   (lambda () (values x ...))))))
          (lambda () (values init ...)))
       (lambda (x ...) expr)))))      
```

## Examples for procedures acting on enumerators

```scheme
(define enumerator->list
  (lambda (enum)
    (assert (procedure? enum))
    (reverse (enumerator->reversed-list enum))))

(define enumerator->reversed-list
  (lambda (enum)
    (assert (procedure? enum))
    (enum (lambda (acc el) (cons el acc)) '()))
    
(define enumerator->generator
  (lambda (enum)
    (assert (procedure? enum))
    (make-coroutine-generator
      (lambda (yield)
        (enum-for-each yield enum)))))
```

## Examples for procedures mapping enumerators

```scheme
(define edrop
  (lambda (k)
    (assert (and (integer? k) (exact? k) (not (negative? k))))    
    (lambda (enum)
      (assert (procedure? enum))
      (lambda (proc nil)
        (assert (procedure? proc))
        (enumerator-for ((k k) (acc nil))
            ((el* enum)
             (if (zero? k)
                 (values 0 apply proc acc el*))
                 (values (- k 1) acc)))
          acc))))
```
