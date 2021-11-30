---
layout: default
---

Baguette is a Lisp->Script compiler. It tries to generate efficient code, and avoid "boilerplate code", at all cost. No types. No user-friendly error messages. Also not finished yet, so don't deploy to mainet.

# Similar to sCrypt

Build your contract the same way, with public functions and everything you already know about.

Two equivalent contracts:

```C
contract Test {
  public function(int a, int b) {
    require(a == b);
  }
}
```

```C
'(public (a b)
    (= a b))
```

# Produced code is efficient

Actually even my CI checks the code produced on simple examples is minimal, but here is some examples.

You need to collect memory manually. Tired of useless `OP_DROPs` and `OP_NIPs` ?  
Me too. So use `(destroy var)`.  
For instance, if `var` is at the top of the stack, `(+ 1 (destroy a))` will produce `OP_1ADD`.

So because you need to collect memory, the code I wrote above is wrong, the correct version is:  
(which just compiles to `OP_EQUAL`, remember it tries to be efficient)

```C
'(public (a b)
    (= (destroy a) (destroy b)))
```

Also you can freely write assembly and modify the stack when you write code. However it's not all user-friendly. But I offer free support. Feel free to give it a quick try.

# It can really get insanely efficient

So our goal is to check if two vars are equal. The code is something like

```C
(define (is-equal a b)
  `(= (destroy ,a) (destroy ,b)))
```

Funny enough you can write something like this:

```C
(define (is-equal f s)
  (if (and (number? f) (number? s))
    (if (= f s) "OP_TRUE" "OP_FALSE")
    `(= (destroy ,f) (destroy ,s))))
```

So when called with only numbers, it'll compile to the result.  
`(is-equal var1 var2)` will move both variable to the top of the stack, and call `OP_EQUAL`.  
`(is-equal int1 int2)` compute the result during the compilation, and only push either `OP_TRUE` or `OP_FALSE`.

You can also specialize if only one of the arguments is known. If you check if you are equal to `OP_0`, you are iif you are `OP_0`, so just call `OP_NOT`.

# Profiling

Not sure what is expensive in your program ? Profile it.

```C
(profile-function
  '(public (a b)
      (define c (call checksigverify (a b))
      (define d (call buildOutput (a b))))
```

Here is the result

```
> racket test.rkt
4   opcodes ================> (define c (call checksigverify (a b)))
  4   opcodes ==============>           (call checksigverify (a b))
82  opcodes ================> (define d (call buildOutput (a b)))
  82  opcodes ==============>           (call buildOutput (a b))
```

No surprise here, checksigverify is an opcode but buildOutput isn't. But here is a more interesting example.

```C
(profile-function
  '(public (tx-arg amount-arg)
    (call pushtx (tx-arg))
    (define scriptCode (call getScriptCode (tx-arg)))
    (define counter (call bin2num ((bytes-get-last scriptCode 1))))
    (define scriptCode_ (+bytes (bytes-delete-last (destroy scriptCode) 1) (+ 1 (destroy counter))))
    (define newAmount (call num2bin ((destroy amount-arg) 8)))
    (define output (call buildOutput ((destroy scriptCode_) (destroy newAmount))))
    (= (call hash256 ((destroy output))) (call hashOutputs ((destroy tx-arg))))
))
```

Which outputs

```
> racket test.rkt
78  opcodes ================> (call pushtx (tx-arg))
90  opcodes ================> (define scriptCode (call getScriptCode (tx-arg)))
  90  opcodes ==============>                    (call getScriptCode (tx-arg))
9   opcodes ================> (define counter (call bin2num ((bytes-get-last scriptCode 1))))
  9   opcodes ==============>                 (call bin2num ((bytes-get-last scriptCode 1)))
    8   opcodes ============>                                (bytes-get-last scriptCode 1)
11  opcodes ================> (define scriptCode_ (+bytes (bytes-delete-last (destroy scriptCode) 1) (+ 1 (destroy counter))))
  11  opcodes ==============>                     (+bytes (bytes-delete-last (destroy scriptCode) 1) (+ 1 (destroy counter)))
    8   opcodes ============>                             (bytes-delete-last (destroy scriptCode) 1)
      1   opcodes ==========>                                                (destroy scriptCode)
    1   opcodes ============>                                                                        (+ 1 (destroy counter))
      0   opcodes ==========>                                                                             (destroy counter)
3   opcodes ================> (define newAmount (call num2bin ((destroy amount-arg) 8)))
  3   opcodes ==============>                   (call num2bin ((destroy amount-arg) 8))
    1   opcodes ============>                                  (destroy amount-arg)
79  opcodes ================> (define output (call buildOutput ((destroy scriptCode_) (destroy newAmount))))
  79  opcodes ==============>                (call buildOutput ((destroy scriptCode_) (destroy newAmount)))
    1   opcodes ============>                                   (destroy scriptCode_)
    0   opcodes ============>                                                         (destroy newAmount)
17  opcodes ================> (= (call hash256 ((destroy output))) (call hashOutputs ((destroy tx-arg))))
  1   opcodes ==============>    (call hash256 ((destroy output)))
    0   opcodes ============>                   (destroy output)
  14  opcodes ==============>                                      (call hashOutputs ((destroy tx-arg)))
    0   opcodes ============>                                                         (destroy tx-arg)
```

Hard to guess which part is the most expensive without this profiling.

(Yes the counter contract works, push_tx too, I check myself [with this](https://replit.com/@frenchfrog42/Counter) when I push code, so you can check too).

# No boilerplate code

Keeping state is boring. So I wrote a library for this. Not 100% perfect, but works for now.

Here is the counter contract with it:

```C
(define compteur-state (vyper-create-final
      '(compteur)
      '((public () (modify compteur (+ 1 compteur))))))
```

And of course instead of `(compteur)`, you can use whatever list, each element will be a state variable. [Yes it works too](https://replit.com/@frenchfrog42/Counter).

This is just an example, using `(destroy var)` everywhere when you don't care about size is boring too, so I wrote a lib too. Transforming source code in Lisp is very easy, so is eliminating boilerplate code. Here is an example of use:

```C
(garbage-collector
  '(public (a b) (= a b))
#f)
```

Which produces

```C
'(public (a b) (= a b) "OP_TOALTSTACK" (cons (drop a) (drop b)) "OP_FROMALTSTACK")
```

Which then compiles to `OP_OVER OP_OVER OP_EQUAL OP_TOALTSTACK OP_NIP OP_DROP OP_FROMALTSTACK`. Not very efficient I know. At least it works :/

You got the idea, if you have a repetitive pattern in your code, it's very easy to get rid of it.

# Try Baguette here

Edit the file `test.rkt` and play with it! Execute `racket test.rkt` to compile. Or press the green arrow. If it's laggy it's replit fault, not mine, in your IDE it won't be I promise.

<iframe frameborder="0" width="100%" height="500px" src="https://replit.com/@frenchfrog42/Embed?embed=true"></iframe>

# New to Baguette ?

[Here is my documentation](http://replit-docs.frenchfrog42.repl.co).
