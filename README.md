# Functional language interpreter

This repo contains an interpreter, written in OCaml, for a small didactic functional language with [static scoping](https://en.wikipedia.org/wiki/Scope_(computer_science)#Lexical_scope_vs._dynamic_scope) and [dynamic type checking](https://en.wikipedia.org/wiki/Type_system#Dynamic_type_checking_and_runtime_type_information).

This was the second project for the *Programmazione II* (Programming Languages) course; the assignment was to extend the functional language presented in class with **tuples of expressions** and with two ways of **combining functions**: pipelines (`Pipe`) and iterated application (`ManyTimes`).

## Repository contents

- [`interprete_funzionale.ml`](https://github.com/matteogiorgi/funint/blob/master/interprete_funzionale.ml): the whole interpreter (environment, abstract syntax, type checker, semantics) and its test suite
- [`specifiche_interprete.pdf`](specifiche_interprete.pdf): the original assignment text (in Italian)
- [`relazione_interprete.pdf`](relazione_interprete.pdf): the project report, describing the main design choices (in Italian)




## Running it

There is no build system: the file is meant to be loaded into the OCaml toplevel, which evaluates every test and prints its result.

```sh
ocaml
```

```ocaml
#use "interprete_funzionale.ml";;
```

Each test is followed in the source by a comment with its expected output, so you can compare the toplevel output against it. To evaluate your own programs, call `sem` on an expression and an empty environment:

```ocaml
sem (Sum (Eint 1, Eint 2), emptyenv Unbound);;
(* - : eval = Int 3 *)
```




## The language

### Abstract syntax

Programs are written directly as values of the `exp` type (there is no parser):

```ocaml
type exp =
  | Eint of int | Ebool of bool                  (* constants *)
  | Den of ide                                   (* identifier *)
  | Prod of exp * exp | Sum of exp * exp
  | Diff of exp * exp | Minus of exp             (* integer arithmetic *)
  | Eq of exp * exp | Iszero of exp              (* integer comparisons *)
  | Or of exp * exp | And of exp * exp | Not of exp
  | Ifthenelse of exp * exp * exp
  | Let of ide * exp * exp                       (* local declaration *)
  | UFun of ide * exp | UAppl of exp * exp       (* unary functions and their application *)
  | Fun of ide list * exp | Appl of exp * exp list   (* n-ary functions and their application *)
  | Rec of ide * exp                             (* recursive functions *)
  | Etup of tuple                                (* tuple of expressions *)
  | Pipe of tuple                                (* composition of unary functions *)
  | ManyTimes of int * exp                       (* n-fold application of a unary function *)
and ide = string
and tuple = Nil | Seq of exp * tuple
```

Compared to the base language seen in class, the project adds recursive functions (`Rec`), separate constructors for unary and n-ary functions (`UFun`/`UAppl` and `Fun`/`Appl`), and the three constructors required by the assignment: `Etup`, `Pipe` and `ManyTimes`.


### Values

```ocaml
type eval =
  | Int of int
  | Bool of bool
  | Unbound                 (* value of an unbound identifier *)
  | Funval of efun          (* closure: function expression + declaration environment *)
  | TupVal of eval list     (* evaluated tuple *)
and efun = exp * eval env
```

The only new value is `TupVal`, which tuples evaluate to. Pipelines and iterations don't need a value of their own: they evaluate to ordinary closures (`Funval`) of unary functions.


### Semantics of the new constructs

- **`Etup (Seq (e1, Seq (e2, ... Nil)))`** evaluates every element, left to right, and returns `TupVal [v1; v2; ...]`.
- **`Pipe (Seq (f1, Seq (f2, ... Seq (fm, Nil))))`** returns the unary function that, applied to `e`, behaves like `fm (... f2 (f1 e) ...)`, like a shell pipeline. The elements can be `UFun` literals, nested `Pipe`s, `ManyTimes`, or any expression that evaluates to a unary closure (for instance an identifier bound to a function). The empty pipe `Pipe Nil` is the identity function.
- **`ManyTimes (n, f)`** returns the function that applies `f` `n` times. It's equivalent to a `Pipe` with `n` copies of `f`. `n` must be at least 1.




## Implementation

### Environment

The environment is a function `string -> 't`, implemented in the `Funenv` module:

- `emptyenv v`: the empty environment, mapping every identifier to the bottom value `v` (`Unbound` in practice)
- `applyenv (r, i)`: looks up identifier `i` in `r`
- `bind (r, i, v)`: extends `r` with the binding `i -> v`
- `bindlist (r, il, vl)`: extends `r` with a list of bindings, raising `WrongBindlist` if the lengths don't match

The file also contains `Listenv`, an alternative implementation of the same `ENV` signature based on association lists. It isn't used: the interpreter opens `Funenv`.


### Static scoping

Function declarations (`UFun`, `Fun`) evaluate to closures that capture the environment they were declared in, and application evaluates the body in that environment extended with the parameters. Recursive functions (`Rec`) build their closure environment as a fixed point, so the function can refer to itself by name.


### Dynamic type checking

`typecheck (t, v)` checks whether value `v` has type `t`, where `t` is one of `"int"`, `"bool"`, `"fun"` or `"tup"`. Each primitive operation (`plus`, `diff`, `mult`, `equ`, `minus`, `iszero`, `et`, `vel`, `non`) checks the types of its operands at runtime and raises `Failure "type error"` if they're wrong. Conditionals require a boolean guard (`"nonboolean guard"`), and applying something that isn't a function raises `"attempt to apply a non-functional object"`.


### Pipelines through substitution

A `Pipe` is turned into a single `UFun` whose body is the composition of the bodies of the functions in the tuple. Composition is done syntactically by `substitute`, which walks an expression and replaces the free occurrences of a parameter with another expression: the parameter of each function is replaced with the body of the function before it in the pipeline. For example:

```ocaml
Pipe (Seq (UFun ("x", Prod (Eint 2, Den "x")),
           Seq (UFun ("x", Sum (Eint 1, Den "x")), Nil)))
(* evaluates to  Funval (UFun ("x", Sum (Eint 1, Prod (Eint 2, Den "x"))), ...) *)
```

`ManyTimes (n, f)` works the same way, composing the body of `f` with itself `n` times.

The tricky part is name clashes: a later function in the pipeline may refer to a free variable that has the same name as the parameter of the final composed function. Without care, that free variable would be captured by the parameter after substitution. In this example, the `x` in the second function should refer to the outer `x = 10`, not to the pipe's argument:

```ocaml
Let ("x", Eint 10,
  Let ("p", Pipe (Seq (UFun ("x", Sum (Den "x", Eint 1)),
                  Seq (UFun ("y", Sum (Den "y", Den "x")), Nil))),
    UAppl (Den "p", Eint 100)))
(* evaluates to Int 111, i.e. (100 + 1) + 10 *)
```

To handle this, such occurrences are first renamed to a reserved placeholder identifier, `__x`, and the closure returned by the pipe binds `__x` to the value the clashing identifier had where the `Pipe` was declared. Because of this, `__x` is reserved: using it as an identifier inside a pipeline raises `Failure "__x not allowed as ide"`.




## Tests

The bottom of [`interprete_funzionale.ml`](interprete_funzionale.ml) contains 15 tests, each followed by its expected result:

| Test | What it covers                                                    | Expected result                                                    |
| ---  | ---                                                               | ---                                                                |
| 1    | Recursive factorial with `Rec` and `Fun`                          | `Int 720`                                                          |
| 2    | Tuple of constants                                                | `TupVal [Int 2; Int 3]`                                            |
| 3    | `Pipe` of two functions, returned as a closure                    | `Funval (UFun ("x", Sum (Eint 1, Prod (Eint 2, Den "x"))), <fun>)` |
| 4    | Applying a `Pipe`: `(2 * 3) + 1`                                  | `Int 7`                                                            |
| 5    | Tuple containing an identifier                                    | `TupVal [Int 10; Int 33]`                                          |
| 6    | `ManyTimes (10, x - 1)` applied to 100                            | `Int 90`                                                           |
| 7    | `Pipe` of functions with different parameter names                | `Int 10`                                                           |
| 8    | `Pipe` where a function body shadows its own parameter with `Let` | `Int 1`                                                            |
| 9    | Name clash between a free variable and the pipe parameter         | `Int 111`                                                          |
| 10   | `Pipe` returning a closure that uses the `__x` placeholder        | `Funval (UFun ("w", Sum (Den "__x", Den "x")), <fun>)`             |
| 11   | `Pipe` containing a function referred to by name                  | `Int 33`                                                           |
| 12   | `Pipe` whose function calls a closure with a clashing name        | `Int 33`                                                           |
| 13   | Static scoping: a later `Let` doesn't affect a closure            | `Int 13`                                                           |
| 14   | Reserved `__x` used as a parameter                                | `Failure "__x not allowed as ide"`                                 |
| 15   | Reserved `__x` used as a free variable                            | `Failure "__x not allowed as ide"`                                 |
