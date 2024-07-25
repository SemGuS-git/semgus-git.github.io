## Programs with Loops in SemGuS

One differentiator of SemGuS is the ability to encode programs with loops. In this tutorial, we will walk
through defining the syntax and semantics of a simple looping SemGuS problem. In particular, we want a
function:
```c
multiply(int x, int y) { ... }
```
that takes two integers and multiplies them together---but by using repeated addition.

> Note: we are _not_ synthesizing an SMT expression or function; rather, in this tutorial, we are synthesizing something more like a C-style function that is called functionally but is implemented with imperative statements, including loops.

### Defining a universe of terms

Our first step is to define a universe of terms; that is, an unrestricted grammar of all terms that our program
will be defined with. We will further restrict this grammar later.

```
F ::= fn(x,y) { S; return E }
S ::= x ← E | y ← E | r ← E | while B do { S } | S; S | noop
E ::= x | y | r | E + E | E - E | 0 | 1
B ::= E < E
```

This is a simple imperative grammar that defines a function `fn(int x, int y) -> int`. Inside the 
function body, the `S` non-terminal defines assignment statements for `x`, `y`, and a register `r`, 
a composition operator `;`, an empty statement `noop`, and a `while` loop statement. Assignments,
the loop condition, and the function return statement all take an integer expression `E`.

**Example**: Summing `x` and `y`. A program that returns the sum of `x`
and `y` does not need any body statements, just the return:
```c
fn(x,y) { noop; return x + y }
```

**Example**: `x`-sign is `y`-parity. A program that returns `x` if `y` is even and `-x` if `y` is odd.
```c
fn(x,y) {
    r ← x;
    while 0 < y do {
        y ← y - 1;
        r ← (x - r) - x
        // r oscillates between +/- x
    };
    return r
} 
```

To encode this universe of terms in a SemGuS file, we use the `declare-term-types` command, which is similar to the `declare-datatypes` command native to SMT-LIB2. Each non-terminal
maps to an algebraic datatype, and each production maps to a constructor for the given datatype. The grammar above, when converted to the SemGuS format, looks like:
```smt
(declare-term-types
 ;; Nonterminals
 ((F 0) (S 0) (E 0) (B 0))
 
;; Productions
 ((($function S E)) ; F
  (($x<- E)         ; S
   ($y<- E)
   ($r<- E)         
   ($while B S)
   ($seq S S)
   ($noop))
  (($r)             ; E
   ($0)
   ($1)
   ($x)
   ($y)
   ($+ E E)
   ($- E E))
  (($< E E))))      ; B
```
By convention, term type constructors are prefixed with `$`, simply to 
reduce namespace collisions; however, this is not required. The two 
example programs can thus be expressed as SMT expressions constructing a
tree of term types:
```smt
; Sum x and y
($function $noop ($+ $x $y))

; x-sign is y-parity
($function
  ($seq ($r<- $x)
        ($while ($< $0 $y)
            ($seq ($y<- ($- $y $1))
                  ($r<- ($- ($- $x $r) $x)))))
  $r)
```

### Semantics as CHCs
So far, we have specified the *syntax* of a program we could synthesize, but not its *semantics*. In SyGuS, for example, the syntax and semantics are equivalent; however, in SemGuS, we must imbue our 
productions with semantic meaning ourselves. 

In the SemGuS format, CHCs are encoded as `match` expressions over term types, with each CHC head getting a separate function. Each CHC head corresponds to a unique non-terminal, but a given non-terminal may have many CHC heads, depending on the desired problem semantics. The CHC head's signature contains the term being matched, the input tuple, and the output tuple. The outline of our example grammar's semantics looks like:
```smt
(define-funs-rec
    ;; CHC heads
    ((F.Sem ((t F) (x Int) (y Int) (ret Int)) Bool)
     (S.Sem ((t S) (xi Int) (yi Int) (ri Int) (xo Int) (yo Int) (ro Int)) Bool)
     (E.Sem ((t E) (xi Int) (yi Int) (ri Int) (out Int)) Bool)
     (B.Sem ((t B) (xi Int) (yi Int) (ri Int) (out Bool)) Bool))
    
    ;; Bodies
    ((! (match t (...F productions...))
        :input (x y) :output (ret))
    
    ; Statements
     (! (match t (...S productions...))
        :input (xi yi ri) :output (xo yo ro))
     
     ; Int expressions
     (! (match t (...E productions...))
        :input (xi yi ri) :output (out))

     ; Boolean expressions
     (! (match t (...B productions...))
        :input (xi yi ri) :output (out))))
```

Our example grammar is interpreted with standard semantics as expected; however, some notes are warranted. Since we are using imperative semantics, each production takes an input state tuple of `(x, y, r)`. Productions from `S` return output state tuples `(x', y', r')`, while 
the remaining productions return a single expression value.

**The Expression Productions.** Each expression production follows the same form of calling the semantics for child productions (if any) and then returning the output of the expression function. For example, `$r` is simply:
```smt
($r (= out ri)) ; <-- Set the output variable to r from the input tuple
```
and `$<` is:
```smt
(($< t1 t2)
 (exists ((o1 Int) (o2 Int))
    (and
      (E.Sem t1 xi yi ri o1) ; <-- Call first child
      (E.Sem t2 xi yi ri o2) ; <-- Call second child
      (= out (< o1 o2)))))   ; <-- Set output to o1 < o2
```

**The Function Production.** The `$function` production handles converting between the desired functional semantics of the program and the internal imperative nature of the program. It creates an initial input state tuple `(x, y, 0)` and passes it to the `S` child, and then passes the output `(xo, yo, ro)` of `S` to the return expression `E` for a final output value. Encoded in SemGuS, this is a match:
```smt
(($function tbody treturn)
 (exists ((xo Int) (yo Int) (ro Int))
    (and
      (S.Sem tbody x y 0 xo yo ro)    ; <-- Call the statement body
      (E.Sem treturn xo yo ro ret)))) ; <-- Return the final expression
```

**The Assignment Productions.** The assignment productions (e.g., `$r<-`) simply update the output state with the new value of the given variable and pass the rest through. For assigning, e.g., `r`, this looks like:
```smt
(($r<- te)
(exists ((out Int))
   (and
     (E.Sem te xi yi ri out) ; <-- Call the expression child
     (= xo xi)               ; <-- Pass x through
     (= yo yi)               ; <-- Pass y through
     (= ro out))))           ; <-- Update r with the new value
```

**The Loop Production.** The `$while` loop production has two separate CHCs: one for the base case (`B` evaluates to false), and one for the recursive case (`B` evaluates to true and the body runs again). Both cases are encoded as two alternatives for the `$while` production:
```smt
(($while tb ts)
 ; Recursive case
 (exists ((b Bool) (x1 Int) (y1 Int) (r1 Int))
    (and
      (B.Sem tb xi yi ri b)         ; <-- Evaluates the condition
      (= b true)                    ; <-- Guard: checks the condition
      (S.Sem ts xi yi ri x1 y1 r1)  ; <-- Calls the statement body
      (S.Sem t x1 y1 r1 xo yo ro))) ; <-- Calls the $while term again
 ; Base case
 (exists ((b Bool))
    (and 
      (B.Sem tb xi yi ri b)         ; <-- Evaluates the condition
      (= b false)                   ; <-- Guard: checks the condition
      (= xo xi)                     ; <-- Noop: copy x
      (= yo yi)                     ; <-- Noop: copy y
      (= ro ri))))                  ; <-- Noop: copy r
```

### The Synthesis Task
Similar to SyGuS, we use the `synth-fun` task to declare what we are attempting to synthesize. However, the SemGuS `synth-fun` specifies the *term* to be synthesized (as a datatype tree). Here, the basic form is:
```smt
(synth-fun mul () F)
```
We want to synthesize a function named `mul` that takes no arguments and returns a value of type `F`; that is, a tree of terms rooted at the non-terminal `F`. This declaration can optionally be augmented with a further-restricted grammar, in the same format as in SyGuS. In this case, we massage the grammar a little to make synthesis more tractable by:
* removing `$noop`
* restricting `B` to use literals
* making `$seq` only right-recursive
* not allowing `$+` and `$-` to recurse
* forcing a loop at the top-level of the function body

```smt
(synth-fun mul () F
           ((F F) (Loop S) (Seq S) (Inner S) (Op E) (Val E) (B B))
           ((F F (($function Loop Val)))
            (Loop S (($while B Seq)))
            (Seq S (($seq Inner Seq) Inner))
            (Inner S (($x<- Op) ($y<- Op) ($r<- Op)))
            (Op E (($+ Val Val) ($- Val Val) Val))
            (Val E ($0 $1 $x $y $r))
            (B B (($< Val Val)))))
```
As SemGuS solvers mature, we hope to not need these restrictions in the future!

### Constraints
We will use programming-by-example (PBE) constraints, specified as a relation over the initial CHC head with `mul` as the term.
```smt
(constraint (F.Sem mul 0 0 0))   ; 0 * 0 = 0
(constraint (F.Sem mul 1 1 1))   ; 1 * 1 = 1
(constraint (F.Sem mul 2 2 4))   ; 2 * 2 = 4
(constraint (F.Sem mul 3 3 9))   ; 3 * 3 = 9
(constraint (F.Sem mul 5 3 15))  ; 5 * 3 = 15
(constraint (F.Sem mul 3 4 12))  ; 3 * 4 = 12
```
 These examples show that we are looking for a program that will multiply `x` and `y`. More specifically, these constraints state that we are looking for a value of the term `mul` such that the relation `(F.Sem mul x y o)` is true for all `x`, `y`, and `o` such that `o = x*y`, with specific examples of `x`, `y`, and `o` given as constraints.

### Synthesizing
The final command in a SemGuS file is the `check-synth` command, which tells the solver to find the solution!
```smt
(check-synth)
```

Given the above constraints, the solution to this synthesis problem can be found by the bottom-up enumerator in `ks2` [[link]](https://github.com/kjcjohnson/ks2-mono) in fewer than 10 seconds. Run the following and see:
```bash
$ ks2 solve loops.sem -s 'bue'
loops.sem:
; <...trimmed...>
(
  (define-fun mul () F
      ($function
          ($while ($< $0 $y)
                  ($seq ($y<- ($- $y $1))
                        ($r<- ($+ $r $x))))
          $r))
)
```
This is the program we were looking for. Nice!

### Full Example SemGuS File
[Download the full example here.](loops.sem)

```smt
;;;;
;;;; loops.sem - multiply via a loop
;;;;

;;;
;;; Term types
;;;
(declare-term-types
 ;; Nonterminals
 ((F 0) (S 0) (E 0) (B 0))
 
;; Productions
 ((($function S E))  ; F
  (($x<- E)          ; S
   ($y<- E)
   ($r<- E)         
   ($while B S)
   ($seq S S)
   ($noop))
  (($r)              ; E
   ($0)
   ($1)
   ($x)
   ($y)
   ($+ E E)
   ($- E E))
  (($< E E))))       ; B

;;;
;;; Semantics
;;;
(define-funs-rec
    ;; CHC heads
    ((F.Sem ((t F) (x Int) (y Int) (ret Int)) Bool)
     (S.Sem ((t S) (xi Int) (yi Int) (ri Int) (xo Int) (yo Int) (ro Int)) Bool)
     (E.Sem ((t E) (xi Int) (yi Int) (ri Int) (out Int)) Bool)
     (B.Sem ((t B) (xi Int) (yi Int) (ri Int) (out Bool)) Bool))
    
    ;; Bodies
    ((! (match t
               ((($function tbody treturn)
                 (exists ((xo Int) (yo Int) (ro Int))
                         (and
                          (S.Sem tbody x y 0 xo yo ro)
                          (E.Sem treturn xo yo ro ret))))))
        :input (x y) :output (ret))
    
    ; Statements
     (! (match t
               ((($x<- te)
                 (exists ((out Int))
                         (and
                          (E.Sem te xi yi ri out)
                          (= xo out)
                          (= yo yi)
                          (= ro ri))))
                (($y<- te)
                 (exists ((out Int))
                         (and
                          (E.Sem te xi yi ri out)
                          (= xo xi)
                          (= yo out)
                          (= ro ri))))
                (($r<- te)
                 (exists ((out Int))
                         (and
                          (E.Sem te xi yi ri out)
                          (= xo xi)
                          (= yo yi)
                          (= ro out))))
                
                (($seq t1 t2)
                 (exists ((x1 Int) (y1 Int) (r1 Int))
                         (and
                          (S.Sem t1 xi yi ri x1 y1 r1)
                          (S.Sem t2 x1 y1 r1 xo yo ro))))
                (($noop)
                 (and
                  (= xi xo)
                  (= yi yo)
                  (= ri ro)))
                (($while tb ts)
                 (exists ((b Bool) (x1 Int) (y1 Int) (r1 Int))
                         (and
                          (B.Sem tb xi yi ri b) 
                          (= b true)
                          (S.Sem ts xi yi ri x1 y1 r1)
                          (S.Sem t x1 y1 r1 xo yo ro)))
                 (exists ((b Bool))
                         (and 
                          (B.Sem tb xi yi ri b)
                          (= b false)
                          (= xo xi) (= yo yi) (= ro ri))))))
        :input (xi yi ri) :output (xo yo ro))
     
     ; Int expressions
     (! (match t
               (($0 (= out 0))
                ($1 (= out 1))
                ($x (= out xi))
                ($y (= out yi))
                ($r (= out ri))
                (($+ t1 t2)
                 (exists ((o1 Int) (o2 Int))
                         (and
                          (E.Sem t1 xi yi ri o1)
                          (E.Sem t2 xi yi ri o2)
                          (= out (+ o1 o2)))))
                (($- t1 t2)
                 (exists ((o1 Int) (o2 Int))
                         (and
                          (E.Sem t1 xi yi ri o1)
                          (E.Sem t2 xi yi ri o2)
                          (= out (- o1 o2)))))))
        :input (xi yi ri) :output (out))

     ; Bool expressions
     (! (match t
               ((($< t1 t2)
                 (exists ((o1 Int) (o2 Int))
                         (and
                          (E.Sem t1 xi yi ri o1)
                          (E.Sem t2 xi yi ri o2)
                          (= out (< o1 o2)))))))
        :input (xi yi ri) :output (out))))

;;;
;;; Function to synthesize - a term rooted at F
;;;

(synth-fun mul () F
           ((F F) (Loop S) (Seq S) (Inner S) (Op E) (Val E) (B B))
           ((F F (($function Loop Val)))
            (Loop S (($while B Seq)))
            (Seq S (($seq Inner Seq) Inner))
            (Inner S (($x<- Op) ($y<- Op) ($r<- Op)))
            (Op E (($+ Val Val) ($- Val Val) Val))
            (Val E ($0 $1 $x $y $r))
            (B B (($< Val Val)))))

;;;
;;; Constraints - examples
;;;
(constraint (F.Sem mul 0 0 0))
(constraint (F.Sem mul 1 1 1))
(constraint (F.Sem mul 2 2 4))
(constraint (F.Sem mul 3 3 9))
(constraint (F.Sem mul 5 3 15))
(constraint (F.Sem mul 3 4 12))

;;;
;;; Instruct the solver to find swap
;;;
(check-synth)
```
