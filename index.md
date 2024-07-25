
# SemGuS

## Semantics-Guided Synthesis

*Semantics-guided synthesis* (SemGuS) is a framework for specifying arbitrary program synthesis problems. SemGuS allows a user to provide both the syntax and the semantics for the constructs in their language. SemGuS accepts a recursively defined big-step semantics, which allows it, for example, to be used to specify and solve synthesis problems over an imperative programming language that may contain loops with unbounded behavior. The customizable nature of SemGuS also allows synthesis problems to be defined over a non-standard semantics, such as an abstract semantics of the user's choice.

### The SemGuS format

The SemGus Front-End Language [[see manual]](res/semgus-lang.pdf) provides a unified file format for describing synthesis problems.

The first part of a SemGuS file is a *regular tree grammar* that describes the **syntax** language in which we are looking for a program.
In SemGuS, a grammar is declared as a recursive data type as follows.

```lisp
(declare-term-types  
 ;; Term types  ; Declare terms inductively using syntactic constructors
 ((E 0) (B 0))  ; Each term type corresponds to a nonterminal in the default grammar

 ;; Productions
 (
   ( ; Productions for term type E
      ($x)      ; The leading `$`-signs are a naming convention with no special significance.
      ($y) 
      ($0)
      ($1)
      ($+ E E)  ; The $+ production has two child terms of type E
      ($ite B E E)
    )
    ( ; Productions for term type B
      ($t) 
      ($f)
      ($not B)
      ($and B B)
      ($or B B)
      ($< E E)
    )
  )
)
```

The largest section of this file is devoted to defining the **semantics** of our language. The ability to specify an arbitrary semantics is a key defining feature of Semgus.

The semantics of productions are encoded using Constrained Horn Clauses (CHC), which mimic how one would define structured operational semantics rules on paper.

Definining a semantics in SemGuS boils down to defining a relation and what inhabits such a relation.
For example, in the code snippet below `($+($x, $1), 5, 2, 6)` is a valid tuple in the relation `E.Sem` (i.e., evaluating the term `$+($x, $1)` on a state where the value of `x` is 5 and the value of `y` is 2, yields the value 6.

```lisp
(define-funs-rec
  ;; Semantic relations and their types
  (
    (E.Sem ((et E) (x Int) (y Int) (r Int)) Bool)  ; Each relation is declared as a function Sem : (members) -> Bool
    (B.Sem ((bt B) (x Int) (y Int) (r Bool)) Bool) ; The `_.Sem` names are a naming convention with no special significance.
  )
  ;; Bodies
  (
    (! ; The `!` expression attaches attributes to its first argument: in this case, the `:in` and `:out` labels below
      (match et ; E.Sem defined inductively through matching
        (
          ($x (= r x)) ; evaluating $x yields the value of x in the relation
          ($y (= r y)) ; evaluating $y yields the value of x in the relation
  ;;; ...
          (
            ($+ et1 et2)            ; et1 and et2 identify the child terms
            (exists
              ((r1 Int) (r2 Int))   ; r1 and r2 are auxiliary variables
              (and
                (E.Sem et1 x y r1)  ; premise 1
                (E.Sem et2 x y r2)  ; premise 2
                (= r (+ r1 r2))     ; constraint about the value of r in the consequence
              )                     ; the value of ($+ et1 et2) is the value of et1 plus the et2
            )                       ; when evaluated on the same inputs x and y
          )
  ;;; ...
        )
      ) ; end `match et`
      :input (x y) :output (r)      ; Indicate that the value of `r` is intended to be determined
    )                               ; from the values of `x` and `y`.
                                    ; These labels may be unnecessary for some solvers.
    (!
      (match bt
  ;;; ...
      )
    )
  )
)
```

Finally, we can provide a **specification** by giving the name of the function we are trying to synthesize and some *constraints* the synthesized function should satisfy (in this case a few input/output examples, but constraints can also be predicates). 

```lisp
(synth-fun max2() E) ; Use the default grammar rooted at E
(constraint (E.Sem max2 4 2 4)) ; on input x=4 and y=2, max2 should return 4
(constraint (E.Sem max2 2 5 5)) ; on input x=2 and y=5, max2 should return 5
(constraint (E.Sem max2 7 1 7)) ; on input x=7 and y=1, max2 should return 7
(check-synth)
```

### Resources

For a more complete guide to the SemGuS front-end format, see the [[SemGuS Language]](/language) page.

To learn more about SemGuS, you can watch this [[5-minute talk]](talks), read the [[vision paper]](https://cseweb.ucsd.edu/~ldantoni/papers/cav21-keynote.pdf), or read the original [[SemGuS paper]](https://cseweb.ucsd.edu/~ldantoni/papers/popl21.pdf).

If you want to use existing SemGuS solvers or build your own solvers, here are some helpful links:
- The SemGuS **format specification** [[pdf]](res/semgus-lang.pdf)
- SemGuS **benchmarks** [[link]](https://github.com/SemGuS-git/Semgus-Benchmarks) (Contributions welcome!)
- A SemGuS **parser** [[C# code]](https://github.com/SemGuS-git/Semgus-Parser), [[NuGet]](https://www.nuget.org/packages/Semgus.Parser), [[Java bindings code]](https://github.com/SemGuS-git/Semgus-Java), [[jitpack]](https://jitpack.io/#SemGuS-git/Semgus-Java)
- An experimental SemGuS solver based on CHCs [[link]](https://github.com/SemGuS-git/Semgus-Messy)
- CLI tool with basic SemGuS **solvers** [[link]](https://github.com/kjcjohnson/ks2-mono) (Build your solver and let us know you did!)


### Contact Us 
Contact the SemGuS team at this [email](mailto:semgus@office365.wisc.edu).

- Keith Johnson (UW-Madison)
- Charlie Murphy (UW-Madison)
- Jinwoo Kim (UW-Madison)
- Rahul Krishnan (UW-Madison)
- Tom Reps (UW-Madison)
- [Loris D'Antoni](https://cseweb.ucsd.edu/~ldantoni/) (UCSD)

