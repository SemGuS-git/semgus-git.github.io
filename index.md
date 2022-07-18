
# SemGuS

## Semantics-Guided Synthesis

*Semantics-guided synthesis* (SemGuS) is a framework for specifying arbitrary program synthesis problems. SemGuS allows a user to provide both the syntax and the semantics for the constructs in their language. SemGuS accepts a recursively defined big-step semantics, which allows it, for example, to be used to specify and solve synthesis problems over an imperative programming language that may contain loops with unbounded behavior. The customizable nature of SemGuS also allows synthesis problems to be defined over a non-standard semantics, such as an abstract semantics of the user's choice.

### The SemGuS format

The SemGus Front-End Language v2.0.0 [[manual]](res/semgus-lang.pdf) provides a unified file format for describing synthesis problems.

The first part of a SemGuS file is a *regular tree grammar* that describes the **syntax** language in which we are looking for a program.
In SemGuS, a grammar is declared as a recursive data type as follows.

```lisp
(declare-term-types  
 ;; Term types  ; Declare terms inductively using syntactic constructors
 ((E 0) (B 0))  ; Grammar nonterminals

 ;; Productions
 ((($x); Productions for nonterminal E
   ($y); The leading `$`-signs are a naming convention with no special significance.
   ($0)
   ($1)
   ($+ ($+_1 E) ($+_2 E))   ; $+_1 has type E and is the name of the first child
   ($ite ($ite_1 B) ($ite_2 E) ($ite_3 E)))

  (($t) ; Productions for nonterminal B
   ($f)
   ($not ($not_1 B))
   ($and ($and_1 B) ($and_2 B))
   ($or ($or_1 B) ($or_2 B))
   ($< ($<_1 E) ($<_2 E)))))
```

The largest section of this file is devoted to defining the **semantics** of our language. The ability to specify an arbitrary semantics is a key defining feature of Semgus.

The semantics of productions are encoded using Constrained Horn Clauses (CHC), which mimic how one would define structured operational semantics rules on paper.

Definining a semantics in SemGuS boils down to defining a relation and what inhabits such a relation.
For example, in the code snippet below '($+($x, $1), 5, 2, 6)' is a valid tuple in the relation 'E.Sem' (i.e., evaluating the term '$+($x, $1)' on a state where the value of 'x' is 5 and the value of 'y' is 2, yields the value 6.

```lisp
(define-funs-rec
  ;; Semantic relations and their types
  ((E.Sem ((et E) (x Int) (y Int) (r Int)) Bool)
   (B.Sem ((bt B) (x Int) (y Int) (r Bool)) Bool))
  ;; Bodies
  ((! (match et ; E.Sem defined inductively through matching
       (($x (= r x)) ; evaluating $x yields the value of x in the relation
        ($y (= r y)) ; evaluating $y yields the value of x in the relation
  ;;; ...
        (($+ et1 et2)
         (exists ((r1 Int) (r2 Int)) ; r1 and r2 are auxiliary variables
             (and
              (E.Sem et1 x y r1) ; premise 1
              (E.Sem et2 x y r2) ; premise 2
              (= r (+ r1 r2))))) ; constraint about the value of r in the consequence
                                 ; the value of $+ et1 et2 is the value of et1 plus the et2
                                 ; when evaluated on the same inputs x and y
```

Finally we can provide a **specification** by giving the name of the function we are trying to synthesize and some *constraints* the synthesized function should satisfy (in this case a few input/output examples, but constraints can also be predicates). 

```
(synth-fun max2() E)
(constraint (E.Sem max2 4 2 4)) // on input x=4 and y=2, max2 should return 4
(constraint (E.Sem max2 2 5 5)) // on input x=2 and y=5, max2 should return 5
(constraint (E.Sem max2 7 1 7)) // on input x=7 and y=1, max2 should return 7
(check-synth)
```

### Resources

To learn more about SemGuS, you can watch this [[5-minutes video]](talks), read this [[vision paper]](https://pages.cs.wisc.edu/~loris/papers/cav21-keynote.pdf), or read the original [[SemGuS paper]](https://pages.cs.wisc.edu/~loris/papers/popl21.pdf).

If you want to use existing SemGuS solvers or build your own solvers, here are some helpful links
- The SemGuS manual [[pdf]](res/semgus-lang.pdf)
- SemGuS benchmarks [[link]](https://github.com/SemGuS-git/Semgus-Benchmarks) (Send us new benchmarks!)
- A SemGuS parser [[C# code]](https://github.com/SemGuS-git/Semgus-Parser), [[NuGet]](https://www.nuget.org/packages/Semgus.Parser), [[Java bindings code]](https://github.com/SemGuS-git/Semgus-Java), [[jitpack]](https://jitpack.io/#SemGuS-git/Semgus-Java)
- CLI with basic SemGuS solvers [[link]](getting-started-setup) (Build your solver and let us know you did!)


### Contact Us 
Contact the SemGuS team at this [email](mailto:semgus@office365.wisc.edu)

Jinwoo Kim (lead) (UW-Madison), Wiley Corning (UW-Madison), Keith Johnson (UW-Madison), Evan Geng (UW-Madison), Kanghee Park (UW-Madison), Anvay Grover (UW-Madison), Tom Reps (UW-Madison), [Loris D'Antoni](https://pages.cs.wisc.edu/~loris/) (UW-Madison)

