## Walkthrough of a Semgus File

*[Index](getting-started)*  
*[Previous](getting-started-cli)*

In the previous section, we used applied our solvers to a simple Semgus problem: finding an integer expression to compute the maximum of two numbers. We will now explore the contents of this Semgus file, [`max2-exp.sl`](test), to demonstrate the Semgus format.

### Metadata

```lisp
(set-info :format-version "2.0.0")
(set-info :author("Jinwoo Kim" "Keith Johnson" "Wiley Corning"))
(set-info :realizable true)
```

The first few lines of the file contain metadata about the problem. These `set-info` tags may contain arbitrary key-value pairs; they are not intended to be accessed by solvers, but may be used to convey information to other tools (such as parsers or the test system) and to human viewers.

Aside from authorship information and the format version of the file, we also see that this problem is realizable: i.e., that a solution can be found. We might use the latter piece of information to skip unrealizable problems when benchmarking a solver that cannot detect this.

### Term types

We now declare a set of term types. Each term type is associated with one or more productions, which are declared here in the form of syntactic constructor signatures. For example, the `($ite ($ite_1 B) ($ite_2 E) ($ite_3 E))` entry in `E`'s production list indicates that a term of type `E` may be constructed from the `$ite` symbol, one child term of type `B`, and two child terms of type `E`.

```lisp
(declare-term-types
 ;; Term types
 ((E 0) (B 0))

 ;; Productions
 ((($x); E productions
   ($y)
   ($0)
   ($1)
   ($+ ($+_1 E) ($+_2 E))
   ($ite ($ite_1 B) ($ite_2 E) ($ite_3 E)))

  (($t) ; B productions
   ($f)
   ($not ($not_1 B))
   ($and ($and_1 B) ($and_2 B))
   ($or ($or_1 B) ($or_2 B))
   ($< ($<_1 E) ($<_2 E)))))
```

(The leading `$`-signs are a naming convention with no special significance.)

### Semantics

The largest section of this file is devoted to defining the semantics of our language. The ability to specify an arbitrary semantics is a key defining feature of Semgus.

The semantics of each production is encoded as a Horn clause. In particular, we follow the pattern that each syntactic constructor has an associated predicate (the *premise*), which, when true, implies that the program's execution is semantically valid (the *conclusion*).

If a term `t` of type `E` is semantically valid over a set of state variables `a,b,c`, then we say that the tuple `(t, a, b, c)` is an element of the semantic relation associated with `E`.

Our first step in defining the semantics is to declare the semantic relations:

```lisp
(define-funs-rec
    ;; Semantic relations
    ((E.Sem ((et E) (x Int) (y Int) (r Int)) Bool)
     (B.Sem ((bt B) (x Int) (y Int) (r Bool)) Bool))
```

(The `_.Sem` pattern is a naming convention with no special significance.)

Each relation is declared as a Boolean-valued function, which would indicate whether or not a given tuple is in the relation.

Note that each position in the tuple is parametrized both by a type and by a variable name. In addition to defining the type of each relation, these lines also declare the variables that will appear in the 

```
  ;; Bodies
  ((! (match et ; E.Sem definitions
       (($x (= r x))
        ($y (= r y))
```

These next few lines begin a pattern-matching expression on the syntax of an `E`-typed term `et`. Each arm of this match expression corresponds to a single production of `E`, and maps that production's syntax to its semantic predicate.

Consider the line `($y (= r y))`. We can read this to mean that, for any term $et$ whose syntax matches `$y`, 
$(r = y) \Longrightarrow (et, x, y, r) \in E.Sem$.

For a more complex example, let's look at the semantics of `+`:

```
;;; ...
        (($+ et1 et2)
         (exists ((r1 Int) (r2 Int))
             (and
              (E.Sem et1 x y r1)
              (E.Sem et2 x y r2)
              (= r (+ r1 r2)))))
```

This introduces several new concepts. In the first line, we bind the variables `et1` and `et2` to refer to the child terms of `et`. Next, the `exists` line declares a set of auxiliary variables, which appear in the premise but not in the conclusion. In this case, we use each variable to hold the output value of one of the child terms.

Within the `exists` block, we encounter an `and` of several clauses. Each premise is required to take the form of a conjunction of semantic relation instances and SMT formulas. Note that while the clauses may optionally be written in a "chronological" order, as is done here, they together represent a 

For terms $et$ whose syntax matches `($+ et1 et2)`, we therefore have the following CHC:

$$(\exists\ r_1,r_2. (et_1, x, y, r_1) \in E.Sem \land (et_2, x, y, r_2) \in E.Sem \land (r = r_1 + r_2)) \Longrightarrow (et, x, y, r) \in E.Sem$$

More productions follow. At the end of the productions of `E`, we find this line:

```
;;; ...

)) :input (x y) :output (r))
```

These optional attributes apply to the `match` block of `E`, and indicate that $E.Sem$ should be a functional relation from $x$ and $y$ to $r$. Solvers may make use of this information to derive a unique operational semantics from the CHCs; this is the case with the enumerative solver we ran earlier.

### Problem definition

The `synth-fun` command defines a term to be synthesized, its term type, and the grammar from which it may be constructed.

```
(synth-fun max2() E)
```

No explicit grammar is given in this example. This will result in an implicit nonterminal being created for each term type, with access to the full set of that term type's productions.

#### Example-based constraints

The desired behavior of the program is specified in `constraint` blocks. The simplest form of constraint provides concrete input-output examples.

```
(constraint (E.Sem max2 4 2 4))
(constraint (E.Sem max2 2 5 5))
(constraint (E.Sem max2 7 1 7))
```

Constraints may also include universally quantified predicates. The example below exactly specifies the behavior of the $max$ function. However, these constraints may be incompatible with some solvers.

```
(constraint (forall ((x Int) (y Int) (r Int))
    (= 
        (E.Sem max2 x y r)
        (and
            (or (= x r) (= y r))
            (>= r x)
            (>= r y)
        )
    )
))
```

After all constraints have been given, the `check-synth` command indicates that the problem has been completely specified and synthesis should begin.

```
(check-synth)
```

### Tests

*Note: this is an unstable feature and may be subject to change.*

As discussed in the previous section, the Semgus CLI can perform tests to validate that the semantics of a language behaves as intended. Using a `:test` metadata block, a user can manually specify the syntax of a program along with representative input-output pairs.

```
(set-info :test (
    (
        ($< $x $y)      ; Syntax of a program
        (:t 1 2 true)   ; Example behavior
        (:t 12 8 false)
    )
    (
        ($+ ($+ $1 $x) $y)
        (:t 1 2 4)
        (:t 5 3 9)
        (:t 0 0 :any)   ; Example with unspecified output value
    )
))
```

The symbol `:t` refers to the specified term, and output variables populated with `:any` will not be checked.

Similarly, we can write a `:solution` block to define one or more terms that are expected to match the constraints placed on a given `synth-fun` target.

```
(set-info :solution (
    (
        max2							; Name of the synth-fun target
        ($ite ($< $x $y) $y $x)			; Viable solution 1
        ($+ $0 ($ite ($< $x $y) $y $x))	; Viable solution 2
    )
))
```
