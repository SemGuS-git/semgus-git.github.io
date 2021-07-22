
## SemGuS Solvers and Benchmarks

Two solvers for SemGuS problems are initially available: Messy and Bless.

### Messy

Messy is a constraint-based Semgus solver that reduces synthesis into solving a set of CHCs, as described in the [Semantics-Guided Synthesis paper](https://dl.acm.org/doi/abs/10.1145/3434311). One key feature of Messy is that it can also prove unrealizability: that is, identify when synthesis problems do not have a solution.

[View Messy on GitHub.](https://github.com/kjw227/Messy-Release) Questions? [Contact Jinwoo Kim.](mailto:pl@cs.wisc.edu)

### Bless

Bless is an enumerative Semgus solver for example-based problems. It performs standard bottom-up enumeration of terms in the user's language, optionally incorporating observational equivalence reduction to prune the search space. Bless is packaged with an interpreter for Semgus; we provide a command-line interface for both interpretation and synthesis tasks.

[View Bless on GitHub.](https://github.com/SemGuS-git/Semgus-Interpreter) Questions? [Contact Wiley Corning.](mailto:wcorning@wisc.edu)

### Benchmarks

Some preliminary SemGuS benchmarks are initially available, including basic examples and regular expression benchmarks.

[View SemGuS benchmarks on GitHub.](https://github.com/SemGuS-git/Semgus-Benchmarks) Questions? [Contact Wiley Corning.](mailto:wcorning@wisc.edu)
