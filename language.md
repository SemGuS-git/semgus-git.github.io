## Front-End SemGuS Language

The SemGuS front-end language format is based on SMT-LIB2 and inspired by SyGuS. It allows specifying
the grammar of a synthesis problem, semantics for each production, and constraints on the desired program.

[The SemGus Front-End Language v1.0](todo) [TODO: link this]

### Example: `max2` (Expressions)

The following is an example synthesis problem for finding a program that returns the maximum of its 
two arguments. The grammar consists of the following syntax constraints:
```
E ::= x | y | 0 | 1 | E + E | if B then E else E
B ::= true | false | !B | B & B | E < E
```
plus the expected semantics associated with each production. The resultant specification looks as
follows:

```lisp
;; Metadata about the synthesis problem
(metadata :author "Jinwoo Kim")
(metadata :realizable true)

;; Abstract term types.
;; E.Terms will be integer-valued expressions, and B.Terms will be Boolean-valued expressions 
(declare-term-type E.Term)
(declare-term-type B.Term)

;; The synthesis objective: a term named max2
(synth-term max2 E.Term (
  ;;
  ;; Declarations for variables needed in the semantic relations
  ;;
  (declare-var (et et1 et2) E.Term)
  (declare-var (bt bt1 bt2) B.Term)

  (declare-var (x y r r1 r2) Int)
  (declare-var (rb rb1 rb2) Bool)

  ;;
  ;; Declarations for nonterminals, including semantic relation signature
  ;;
  (declare-nt E E.Term (E.Sem (E.Term Int Int Int)))
  (declare-nt B B.Term (B.Sem (B.Term Int Int Bool)))

  ;;
  ;; Finally, the grammar productions and semantic relations.
  ;;
  ;; The first non-terminal, specifying the CHC head (E.Sem et x y r)
  ((E et) (E.Sem et x y r)
  
    ;; Leaves, with a syntax constructor (x.Syn) and a CHC body (= r x)
    (x.Syn (= r x))
    (y.Syn (= r y))
    (0.Syn (= r 0))
    (1.Syn (= r 1))

    ;; Operators, with additional terms as children
    ((+.Syn (E et1) (E et2))
      ;; The CHC body contains the semantic relations for the child terms
      (and
        (E.Sem et1 x y r1)
        (E.Sem et2 x y r2)
        (= r (+ r1 r2))))

    ;; if B then E else E
    ((ITE.Syn (B bt1) (E et1) (E et2))
      ;; Note there are two CHCs bodies for this one.
      ;; One when B is true...
      (and
        (B.Sem bt1 x y rb)
        (= rb true)
        (E.Sem et1 x y r))
      ;; ...and one when B is false.
      (and
        (B.Sem bt1 x y rb)
        (= rb false)
        (E.Sem et2 x y r))))

  ;; The second non-terminal, specifying the CHC head (B.Sem bt x y rb)
  ((B bt) (B.Sem bt x y rb)

     (True.Syn (= rb true))
     (False.Syn (= rb false))
     ((Not.Syn (B bt1))
       (and
         (B.Sem bt1 x y rb1)
         (= rb (not rb1))))
     ((And.Syn (B bt1) (B bt2))
       (and
         (B.Sem bt1 x y rb1)
         (B.Sem bt2 x y rb2)
         (= rb (and bt1 bt2))))
     ((<.Syn (E et1) (E et2))
       (and
         (E.Sem et1 x y r1)
         (E.Sem et2 x y r2)
         (= rb (< r1 r2)))))))

;;
;; We provide an example-guided constraint
(constraint (and (E.Sem max2 4 2 4) (E.Sem max2 2 5 5)))
```

### Example: `max2` (Imperative)
The preceding grammar uses only integer and Boolean expressions, and an equivalent problem could be
written in SyGuS, assuming standard semantics. However, SemGuS is not limited to only grammars of 
expressions; an alternate, imperative grammar is also possible, over the following syntax:
```
S ::= x = E | y = E | c = E | if B then E else E
E ::= x | y | 0 | 1
B ::= true | false | not B | B & B | E < E
```

This time, imperative semantics are specified for the `Start` non-terminal's productions. 

```lisp
(metadata :author "Jinwoo Kim")
(metadata :realizable true)

(declare-term-type S.Term)
(declare-term-type E.Term)
(declare-term-type B.Term)

(synth-term max2i S.Term (
  (declare-var (x y c rx rx1 rx2 ry ry1 ry2 rc rc1 rc2 r r1 r2 v) Int)
  (declare-var (rb rb1 rb2 vb) Bool)
  
  (declare-var (st st1 st2) S.Term)
  (declare-var (et et1 et2) E.Term)
  (declare-var (bt bt1 bt2) B.Term)
  
  (declare-nt S S.Term (S.Sem (Term Int Int Int Int Int Int)))
  (declare-nt E E.Term (E.Sem (E.Term Int Int Int Int)))
  (declare-nt B B.Term (B.Sem (B.Term Int Int Int Bool)))
  
  ;; Same x and y as before, but with an auxiliary variable c.
  ;; Start is over statements, with x, y, and c the input state, and rx, ry, and rc the output state
  ((S st) (S.Sem st x y c rx ry rc)
    ;;
    ;; Assignment operators for x, y, and c
    ((x=.Syn (E et1)) (and
      (E.Sem et x y c v)
      (= rx v)   ; x gets set to the result...
      (= ry y)   ; ...but y and c are unchanged 
      (= rc c)))
    ((y=.Syn (E et1)) (and
      (E.Sem et x y c v)
      (= rx x)
      (= ry v)
      (= rc c)))
    ((c=.Syn (E et1)) (and
      (E.Sem et x y c v)
      (= rx x)
      (= ry y)
      (= rc v)))
    ;;
    ;; If-then-else over statements
    ((sITE.Syn (B tb) (S s1) (S s2)) (and
      (B.Sem tb x y c vb)
      (S.Sem s1 x y c rx1 ry1 rc1)
      (S.Sem s2 x y c rx2 ry2 rc2)
      (= rx (ite vb rx1 rx2))
      (= ry (ite vb ry1 ry2))
      (= rc (ite vb rc1 rc2)))))
  
  ;;
  ;; The remainder of the grammar is essentially the same as the expression-based grammar above
  ;;
  ((E et) (E.Sem et x y c r)
    (x.Syn (= r x))
    (y.Syn (= r y))
    (c.Syn (= r c))
    (0.Syn (= r 0))
    (1.Syn (= r 1))
    ((+.Syn (E et1) (E et2))
      (and
        (E.Sem et1 x y c r1)
        (E.Sem et2 x y c r2)
        (= r (+ r1 r2))))

    ((ITE.Syn (B bt1) (E et1) (E et2))
      (and
        (B.Sem bt1 x y c rb)
        (= rb true)
        (E.Sem et1 x y c r))
      (and
        (B.Sem bt1 x y c rb)
        (= rb false)
        (E.Sem et2 x y c r))))

  ;; The second non-terminal, specifying the CHC head (B.Sem bt x y rb)
  ((B bt) (B.Sem bt x y rb)

     (True.Syn (= rb true))
     (False.Syn (= rb false))
     ((Not.Syn (B bt1))
       (and
         (B.Sem bt1 x y c rb1)
         (= rb (not rb1))))
     ((And.Syn (B bt1) (B bt2))
       (and
         (B.Sem bt1 x y c rb1)
         (B.Sem bt2 x y c rb2)
         (= rb (and bt1 bt2))))
     ((<.Syn (E et1) (E et2))
       (and
         (E.Sem et1 x y c r1)
         (E.Sem et2 x y c r2)
         (= rb (< r1 r2)))))))

;;
;; We need free variables c1 and c2 in our constraints, as we don't care about their ending value
;;
(declare-var (c1 c2) Int)

;;
;; We also don't care about the start value of c, so we set it to zero to start
;;
(constraint (and (S.Sem max2i 4 (- 1) 0 4 (- 1) c1) (S.Sem max2i (- 2) 3 0 3 (- 2) c2)))
```
