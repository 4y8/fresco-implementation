# Fresco implementation
This is the artifact for the paper "Bidirectional Typing with Freezing,
Skeletons, and Ghosts", by Wenhao Tang, Shengyi Jiang, Aghilas Y. Boussaa, Sam
Lindley, and Bruno C. d. S. Oliveira.

This directory contains an implementation of a language with Fresco-based type
inference. We extend the type inference algorithm of Section 4 with modal effect
types and algebraic data types. This implementation also has a web interface
available [online](https://www.normalesup.org/~boussaa/met/playground/).

Any computer supported by OCaml 5.4 should be usable to evaluate this artifact.

## List of claims
1. Section 5 overviews an extension of Fresco to algebraic data types (Section
   5.1) and modal effect types (Section 5.2). The type inference algorithm
   implemented in `lib/type.ml` supports both of these extensions
   which are evaluated in Step 1 of the evaluation instructions.
2. Appendix A states whether some expressions are well-typed. The directory
   `tests/fresco` contains tests for the Fresco results in these tables,
   evaluated in Step 2 of the evaluation instructions.

## Download, installation, and sanity-testing
### Download

Clone this repository.

### Building
This interpreter is written in OCaml and requires OCaml 5.4 or later. The
following command installs a suitable version of the compiler with
[opam](https://opam.ocaml.org/doc/Install.html).
```sh
opam switch create ocaml-base-compiler.5.4.1
```
This artifact uses the `dune` build system and the `menhir`, `bindlib`, and
`multicont` libraries. The following command installs all required packages.
```sh
opam install dune menhir bindlib multicont
```
To build a binary you can then run:
```sh
dune build
```
The produced binary is called `bin/met.exe`. The whole building process
(including installing an ocaml compiler) took less than ten minutes on a Mac M1.

### Using
The following command spawns a REPL accepting definitions and expressions
terminated by `;;`: 
```sh 
dune exec bin/met.exe
```
Optionally, install [`rlwrap`](https://github.com/hanslub42/rlwrap)
and use it for navigation and history:
```sh
rlwrap -- dune exec bin/met.exe
```
The type-checker can take a file as input:
```sh
dune exec -- bin/met.exe <file> 
```

The `--eval` flag, which has to be put before the file's name, runs the 
interpreter on the file. This flags expects a `main` function in the file.

The `--debug` flag prints a tree of the typing rules used for the input file.

### Examples
We can define a `gen` parametric effect and use it as follows with the `iter`
function. This example uses both algebraic data types and effect handlers.
```
type unit = Unit
type list a = Nil | Cons of a, list a
type option a = None | Some of a
type bool = True | False

eff gen [a] = yield : a => unit

val as_list : [](<gen int>(unit -> unit) -> list int)
let as_list f =
  handle f () with
  | return u => Nil
  | yield x r => Cons (x, r ())
  end

val iter : forall a . []((a -> unit) -> list a -> unit)
let iter f l =
  match l with
  | Nil -> ()
  | Cons (hd, tl) -> f hd; iter f tl
  end

val find : forall [a] . []((a -> bool) -> list a -> option a)
let find p xs =
  handle (iter (fun x -> if mask<gen> (p x) then do yield x else ()) xs) with
  | return _ => None
  | yield x k => Some (x)
  end
```

The `tests/pass` directory contains more examples such as a fragment of a Unix
interface (`unix.mle`) or an elaboration algorithm for a dependently-typed
language (`faux-tt.mle`).

## Evaluation instructions
The paper claims that we can extend Fresco to support algebraic data types and
modal effects (claim 1). The command below tests the type-checker on some
programs that use these features.
```sh
./test.sh bin/met.exe tests/
```
It ensures that the type-checker accepts the programs `tests/pass` and rejects
those in `tests/fail`. The command should print `failed 0 tests out of 25`.

The appendix gives a table of expressions well-typed and ill-typed in the Fresco
type system. The command below tests the type-checker tests that the program
that should fail (`tests/fresco/fail`) do fail and that the one that should pass
(`tests/fresco/pass`) do pass.
```sh
./test.sh bin/met.exe tests/fresco
```
Files are named after the corresponding examples in the tables (`f2.mle`
for instance). Files `a10.mle`, `a12.mle`, `a13.mle`, and `b1.mle` group
the corresponding lettered subcases. Most tests use native lists;
`f6.mle` and `f7.mle` use Church-encoded lists. Examples requiring freezing
have separate accepted and rejected tests. Files `beta1.mle` and
`beta3.mle` are additional tests outside the tables. The command should
print `failed 0 tests out of 57`.

A test in a `pass` directory fails when the type-checker rejects it. A test in a
`fail` directory fails when the type-checker accepts it.
## Additional artifact description
### Overview
The `lib` directory contains the core of the language and it contains the
following file:
+ `lexer.mll` implements a lexer
+ `parser.mly` implements a parser
+ `syntax.ml` defines the syntax of the language
+ `context.ml` defines contexts and implements context manipulation
+ `effects.ml` implements basic functions on effects
+ `type.ml` implements type checking
+ `eval.ml` implements an interpreter for the language
+ `pprint.ml` implements pretty printing of types
+ `error.ml` and `errors.ml` implement error messages

### Mapping functions to judgements
We map important functions in `type.ml` to the corresponding judgements in the
paper.
+ `finfer` implements type inference
+ `sk_infer` implements skeleton inference
+ `join_sk`, `join_var`, and `assign` implement consistency
+ `sub` and `sub_flex` implement subtyping
+ `look` implements looking
+ `constr_collect_sub` and `constr_collect_eq` implement constraint collection
+ `solve_eq`, `solve_sub`, and `constr_solve` implement constraint solving
Consistency functions, and the ones that mimic them, also have versions that work on
modalities.
