---
title: "lambda calculus endeavors"
date: 2026-03-22T17:05:00-07:00
tags: ["computer science"]
---

i would NOT survive a drug addiction. ive recently realized that maybe the algorithm really *is* giving me brain damage -- when i finish an extra 6 hours of coding on top of 8 hours at work, eyelids barely holding up, it doesnt even occur to me that the reasonable decision would be to sleep. why of course i gotta make sure i havent missed anything on youtube today, *all* of it

i still distinctly remember my league of legends addiction -- 16 hours a day during that fateful winter break, from 9am to 1am, every single day. the olden days. but at some point league of legends simply lost all its appeal and i never played summoners rift again. i dont know, maybe ive just played all of it, as opposed to algorithm feed which is an infinite treasure trove

thank god i was blessed with the knowledge that your instagram feed can be reset. ive not indulged myself in scrolling instagram reels for more than 10 minutes ever since. such a stark contrast from 11 months ago (ebk, twn)... but of course, youtube is fundamentally different from instagram reels -- it is full of informational documentaries of the highest academic rigor. or so i believed, until recent brief moments of clarity revealed that as i scroll the infinite stream of wisdom, i am strolling down a dark path to hell

while ive digressed since the very beginning of this post, what i meant to get at was that youtube led me back to them tsoding videos -- namely [the one where he takes a look at chibicc](https://www.youtube.com/watch?v=_dVchhnO_KI), which inspired me to write [compilers](https://github.com/hljeong/cc) yet again. i did not forget to pay homage to mr gcc by pulling the pro gamer move of shifting the naming semantics from "c compiler" to "compiler collection". anyway, while the c compiler project was just getting going, youtube decided to interject and recommend me [this other tsoding video where he "implements"? lambda calculus](https://www.youtube.com/watch?v=KuVUfbWoROw). this piqued my interest, and here we are, writing a blog about it after sinking maybe 8 hours into coding it up. unfortunately missed the opportunity to go out for dinner, a big setback for hitting my goal this year of adding 365 new places to beli


## end ramble, begin content

while there are many topics to talk about here (_e.g._, writing c, error reporting, lexing, etc.), id like to focus on lambda calculus and the current design of [`lc`](https://github.com/hljeong/cc/tree/7bcee9ff3ab01b82aed5b5798793f7dbdf7cd4f9/lambda-calculus) ("\[l\]ambda calculus \[c\]ompiler")


### what is lambda calculus

much like any other topic, never in a thousand years would i call myself an expert in lambda calculus -- im barely an expert in calculus. here ill offer my humble understanding of this curious construction, but keep in mind that i am but a blind man in front of an elephant

lambda calculus is a *model of computation*, which wikipedia defines as a model that describes how the output of a mathematical function is computed given an input. one may be more acquainted with the well-famed model of computation called turing machine, which moves around an infinite tape reading and writing 0s and 1s. personally i think of a model of computation as a system of rules operating on constructs, which, if you play with enough, suddenly starts doing math and you might just be able to play league on it (if the system [completes the turing](https://en.wikipedia.org/wiki/Turing_completeness))

lambda calculus is a language of *lambda terms*, defined by the following 3 constructs:
1. a *variable* is a lambda term, _e.g._, `x`, `y`, `long_variable_name_archnemesis_of_mathematicians`
2. a *function*, *binder*, or *abstraction*, `(λx.M)`, where the *parameter* `x` is a variable and the *body* `M` is a lambda term, is a lambda term
   - since `λ` is a pain in the posterior to type, we will use backslash in its stead: `(\x.M)`
3. an *application* `(f x)`, where `f` and `x` are lambda terms, is a lambda term

some more terminology:
- a variable is a *binding site*\* if it is the parameter of a function
  - the binding site may optionally refer to the `\` symbol as well
- a variable `x` is a *reference*\* if it is not a binding site
  - _e.g._, in `\x.y`, `x` is a binding site and `y` is a reference to `x`
- references to `x` in `M` are *bound* in `\x.M`
- a reference `x` that is not bound is said to be *free*
  - _e.g._, in `(\x.x y)`, `x` in the body is bound (to `\x`) while `y` is free
  - note that shadowing may occur: `(\x.((\x.x) x)) x` is a valid lambda term, but it may be clearer written as `(\x1.((\x2.x2) x1)) x0` -- `x0` is free, `x1` shadows `x0`, and `x2` shadows `x1`

\*possibly non-standard -- my own terminology

the one computational rule in lambda calculus is *beta-reduction* (i know not wherefore it is _beta_), or in laymans terms, substitution. a *beta-reducible expression*, or *beta-redex*, is an application of the form `(\x.M) N`, which reduces to the result of substituting every `x` in `M` with `N`. for instance, `(\x.x z x) (y w)` reduces to `(y w) z (y w)` by substituting every `x` in `x z x` with `(y w)`. the jargon for this is `(\x.M) N` -> `M[x := N]`, and for our example: `(\x.x z x) (y w)` -> `(x z x)[x := (y w)]` -> `(y w) z (y w)`. this is also precisely the same substitution we do in algebra: if the function `bag` is defined as `bag(x) = x + z + x`, then `bag(fries + 1) = (fries + 1) + z + (fries + 1)`

a lambda term is in *beta-normal form* if it contains no beta-redexes, _i.e._, there is nothing to reduce. for instance, `(\x.(\y.y) x) ((\z.z) w)` reduces as follows: `(\x.(\y.y) x) ((\z.z) w)` -> `(\x.(\y.y) x) w` -> `(\y.y) w` -> `w`, and `w` is in beta-normal form

a weaker condition than normal form is *weak head normal form*, or *whnf*: a term where the outermost term is not a redex -- _i.e._, a variable, a function, or an application whose left side is not a function. some examples:
- `x`
- `\x.((\y.y) x)`: could be reduced to `\x.x`, but since it is already a function, this is in whnf
- `((\x.x) y) ((\x.x) y)`: could be reduced to `y y`, but since the left side is _not_ a function, this is in whnf

not every lambda term has a normal form. consider the classic example `Ω = (\x.x x)(\x.x x)`, which reduces to itself in one step: `(\x.x x)(\x.x x)` -> `(x x)[x := (\x.x x)]` -> `(\x.x x)(\x.x x)`. it is clear that this will never reduce to an irreducible term

the order in which we reduce a lambda term makes a difference. *applicative order*, or *eager evaluation*, reduces the innermost leftmost redex first, and *normal order* reduces the outermost leftmost redex first. consider the term `(\x.(\y.y)) Ω`. applicative order reduces `Ω` first, which yields itself, leaving `(\x.(\y.y)) Ω` as is, looping forever. normal order reduces the outermost application first: `(\x.(\y.y)) Ω` -> `(\y.y)[x := Ω]` -> `\y.y`, arriving at normal form. in fact, the normalization theorem, which is a corollary of the [church-rosser theorem](https://en.wikipedia.org/wiki/Church%E2%80%93Rosser_theorem), states that if a term has a normal form, normal reduction *will* find it

there are a number of rabbit holes in front of us that i may or may not dive into in future posts, but for now we shall pivot to implementing this calculus


### implementation

we would like to write a program that parses a lambda term and reduces it to nf or whnf, giving up after a specified number of steps. the desired cli interface looks something like:
```sh
# lc <expr> [nf=(nf)|whnf] [max-steps=10]

$ lc '(\x.(\y.y) x) ((\z.z) w)'
w

# only reduce 1 step
$ lc '(\x.(\y.y) x) ((\z.z) w)' nf 1
(\y.y) ((\z.z) w)

$ lc '((\x.x) y) ((\x.x) y)'
y y

# application where left side is not a function (already whnf)
$ lc '((\x.x) y) ((\x.x) y)' whnf
((\x.x) y) ((\x.x) y)

# give up
$ lc '(\x.x x)(\x.x x)'
(\x.x x)(\x.x x)
```

we first lex the input -- _i.e._, parse the input string into a stream of tokens. there are only a few types of tokens: `(`, `)`, `\`, `.`, and identifiers representing variables. for instance, `((\foo.foo) bar) ((\foo.foo) bar)` yields `(`, `(`, `\`, `foo`, `.`, `foo`, `)`, `bar`, `)`, `(`, `(`, `\`, `foo`, `.`, `foo`, `)`, `bar`, and `)`

we then parse the token stream into an ast (abstract syntax tree) with 3 node types corresponding to the 3 constructs:
1. a variable node contains its name as a string.
2. a function node contains a pointer to the parameter (a variable node) and a pointer to the body (any node)
3. an application node contains a pointer to the function (any node) and a pointer to the argument (any node)

a context-free grammar for lambda terms is given by:
```
expr ::= expr atom | atom
atom ::= "(" expr ")" | fun | var
fun  ::= "\" var "." expr
var  ::= ident
```

the `atom` nonterminal is introduced to enforce left-associativity of applications: `f g h` is parsed as `(f g) h` as opposed to `f (g h)`. however, `expr ::= expr atom | atom` is still left-recursive, so we need to iteratively parse `atom`s and wrap them inside application nodes:
```c
Node *parse_expr(void) {
  Node *node = parse_atom();
  Node *arg = NULL;
  while ((arg = parse_atom())) {
    node = new_app_node(/*fun=*/ node, /*arg=*/ arg);
  }
  return node;
}
```

the other rules are rather straightforward to implement:
```c
// check if next token is of given kind
bool match(TokenKind);

// if next token matches given kind, return the token.
// otherwise return NULL
Token *consume(TokenKind);

// if next token matches given kind, return the token.
// otherwise abort parsing
Token *expect(TokenKind);

// atom ::= "(" expr ")" | fun | var
Node *parse_atom(void) {
  // "(" -> "(" expr ")"
  if (consume(TokenKind_LPAREN)) {
    Node *node = parse_expr();
    expect(TokenKind_RPAREN);
    return node;
  }

  // "\" -> fun
  else if (match(TokenKind_BACKSLASH)) return parse_fun();

  // ident -> var
  else if (match(TokenKind_IDENT)) return parse_var();

  else abort("expected expression");
}

// fun ::= "\" var "." expr
Node *parse_fun(void) {
  expect(TokenKind_BACKSLASH);
  Node *var = parse_var();
  expect(TokenKind_DOT);
  Node *body = parse_expr();
  return new_fun_node(/*var=*/ var, /*body=*/ body);
}

// var ::= ident
Node *parse_var(void) {
  const char *name = expect(TokenKind_IDENT)->lexeme;
  return new_var_node(/*name=*/ name);
}
```

our example from before, `((\foo.foo) bar) ((\foo.foo) bar)`, yields the following ast:
```
((\foo.foo) bar) ((\foo.foo) bar)
└─ app
   ├─ fun: ((\foo.foo) bar)
   │        └─ app
   │           ├─ fun: (\foo.foo)
   │           │        └─ fun
   │           │           ├─ param: foo <──┐
   │           │           └─ body:  foo ───┘
   │           └─ arg: bar (free)
   └─ arg: ((\foo.foo) bar)
            └─ app
               ├─ fun: (\foo.foo)
               │        └─ fun
               │           ├─ param: foo <──┐
               │           └─ body:  foo ───┘
               └─ arg: bar (free)
```

after parsing the ast, we can reduce the lambda term. this is fundamentally an iterative process:
1. find a beta-redex in normal order
2. if a redex is found, perform beta-reduction on it
3. if no redex is found, terminate -- we have arrived at normal form
4. repeat for up to the specified maximum number of steps

we need to first find a beta-redex given an ast -- let us assume `beta(node)` handles performing beta-reduction for now:
```c
// perform one step of normal-order beta-reduction on `node`.
// return `node` unchanged if there is nothing to reduce.
// reduce to whnf if `whnf` is true, nf otherwise
Node *reduce_step(Node *node, bool whnf) {
  // variables are not redexes
  if (node->kind == NodeKind_VAR) return node;

  else if (node->kind == NodeKind_FUN) {
    // whnf does not reduce functions
    if (whnf) return node;

    // look for redexes in the function body
    Node *body = reduce_step(node->body, false);
    // a redex is found within the function body.
    // return a new node that points to the new body
    if (body != node->body)
      return new_fun_node(/*var=*/ node->var, /*body=*/ body);

    // no redexes found
    return node;
  }

  else if (node->kind == NodeKind_APP) {
    // immediately beta-reduce outermost applications (normal order)
    if (node->fun->kind == NodeKind_FUN) return beta(node);

    // whnf does not reduce applications whose lhs is not a function
    if (whnf) return node;

    // look for redexes in the function
    Node *fun = reduce_step(node->fun, false);
    if (fun != node->fun)
      return new_app_node(/*fun=*/ fun, /*arg=*/ node->arg);

    // look for redexes in the argument
    Node *arg = reduce_step(node->arg, false);
    if (arg != node->arg)
      return new_app_node(/*fun=*/ node->fun, /*arg=*/ arg);

    return node;
  }

  else assert(false);
}
```

note that we are performing pointer comparisons to detect reductions -- if the returned pointer differs from the input, a reduction took place somewhere inside. this is a purely functional style: data is immutable and new nodes are created on change rather than mutating in place

let us now look at `beta(node)` which performs the actual substitution:
```c
// substitute references to `var` within `node` with `subval`.
// return `node` unchanged if no substitution happens
Node *sub(Node *node, Node *var, Node *subval) {
  assert(var->kind == NodeKind_VAR);

  if (node->kind == NodeKind_VAR) {
    // replace variable references with `subval` if the names match
    return strcmp(node->name, var->name) ? node
                                         : subval;
  }

  else if (node->kind == NodeKind_FUN) {
    Node *body = sub(node->body, var, subval);
    if (body != node->body)
      return new_fun_node(/*var=*/ node->var, /*body=*/ body);
    return node;
  }

  else if (node->kind == NodeKind_APP) {
    Node *fun = sub(node->fun, var, subval);
    Node *arg = sub(node->arg, var, subval);
    if (fun != node->fun || arg != node->arg)
      return new_app_node(/*fun=*/ fun, /*arg=*/ arg);
    return node;
  }

  else assert(false);
}

Node *beta(Node *node) {
  assert(node->kind == NodeKind_APP);
  assert(node->fun->kind == NodeKind_FUN);
  return sub(node->fun->body, node->fun->var, node->arg);
}
```

let us trace through a call to `reduce_step()`. consider the ast from before:
```
((\foo.foo) bar) ((\foo.foo) bar)
└─ app
   ├─ fun: ((\foo.foo) bar)
   │        └─ app
   │           ├─ fun: (\foo.foo)
   │           │        └─ fun
   │           │           ├─ param: foo <──┐
   │           │           └─ body:  foo ───┘
   │           └─ arg: bar (free)
   └─ arg: ((\foo.foo) bar)
            └─ app
               ├─ fun: (\foo.foo)
               │        └─ fun
               │           ├─ param: foo <──┐
               │           └─ body:  foo ───┘
               └─ arg: bar (free)
```

the top-level node is an application whose left side is also an application. if `whnf` were true, the reduction would halt here since the term is already in whnf. however, when reducing to nf, `reduce_step()` recurses into its children looking for redexes. indeed, the left child is a redex: an application of `(\foo.foo)` to `bar`. beta-reduction yields:
```
app                                app
├─ fun: app        ────beta()───> ├─ fun: bar
│       ├─ fun: \foo.foo     ┌──> └─ arg: app
│       └─ arg: bar          │            ├─ fun: \foo.foo
└─ arg: app      ──unchanged─┘            └─ arg: bar
        ├─ fun: \foo.foo
        └─ arg: bar
```

to fully reduce the input term, we simply call `reduce_step()` until it returns the same node or until `max_steps` is reached:
```c
int main(int argc, char **argv) {
  if (argc < 2) errorf("usage: %s <expr> [nf=(nf)|whnf] [max-steps=10]", argv[0]);
  const char *src = argv[1];
  bool whnf = argc >= 3 ? !strcmp(argv[2], "whnf") : false;
  int max_steps = argc >= 4 ? atoi(argv[3]) : 10;

  Token *toks = lex(src);
  Node *ast = parse(toks);
  for (int steps = 0; steps < max_steps; steps++) {
    Node *nxt = reduce_step(ast, whnf);
    if (nxt == ast) break;
    ast = nxt;
  }
  print_expr(ast);
  return 0;
}
```

now consider the reduction `(\x.(\y.x)) z w` -> `(\y.z) w` -> `z`. if we change the name of the free variable `z` to `y`, something peculiar happens: `(\x.(\y.x)) y w` -> `(\y.y) w` -> `w` -- the output becomes the other free variable `w` instead of the expected `y`. this is because substituting `y` for `x` in `(\y.x)` accidentally *captured* the free variable `y` under the inner `\y`, which is semantically incorrect.

there are a couple of standard fixes:
- *alpha-renaming*: two functions that differ only in their bound variable name and corresponding references in the body are considered *alpha-equivalent* -- formally, `\x.M` and `\y.M[x := y]` are alpha-equivalent. _e.g._, `\x.x` and `\y.y` are equivalent. with alpha-renaming, the bound variable of a function is renamed to avoid capturing any free variables in scope before substitution. for instance, `(\y.x)` might be renamed to `(\z.x)` before substituting `y` for `x`, yielding `(\z.y)` -- and the earlier issue goes away: `(\x.(\z.x)) y w` -> `(\z.y) w` -> `y`
- *de bruijn indices*: in *de bruijn notation*, each variable reference is represented as a number denoting how many binders lie between the reference and its binding site. this eliminates bound variable names entirely, so if free variables are named, captures cannot occur. if free variables are instead represented as indices beyond the outermost binder, care need be taken when substituting -- but that is out of scope for this post

our implementation takes a third approach. the precise location in our code where capturing happens is within `sub()`:
```c
if (node->kind == NodeKind_VAR) {
  // replace variable references with `subval` if the names match
  return strcmp(node->name, var->name) ? node
                                       : subval;
}
```

here we perform the substitution whenever a variable's name matches the bound variable name, even if the variable is referring to some free variable. to fix this issue, each variable node additionally carries a pointer `ref` to its binding site and only substitute if a reference's `ref` points to the same node as `var`:
```c
if (node->kind == NodeKind_VAR) {
  // replace variable references pointing to `var` with `subval`
  return node->ref == var ? subval : node;
}
```

to populate `ref` when the node is created, we must find the closest binding site with the same name. to achieve this, each function node carries a pointer `par` to its parent function node, forming a scope chain. when parsing a variable reference, we walk up this chain looking for a binding site whose name matches. if no such binding site is found, `ref` is left as `NULL` -- it will never match any binding site during substitution, effectively making the variable free:
```c
Node *get_var(const char *name) {
  Node *scope = current_scope;
  while (scope) {
    if (!strcmp(scope->var->name, name))
      return new_var_node(/*name=*/ name,         // found binding var node,
                          /*ref=*/  scope->var);  // create a reference
    scope = scope->par;
  }
  return new_var(name, NULL);  // free variable
}

Node *parse_var(const bool binding) {
  const char *name = expect(TokenKind_IDENT)->lexeme;
  if (binding) return new_var_node(/*name=*/ name, /*ref=*/ NULL);
  else         return get_var(name);
}
```

in the actual implementation, free variables create fictitious binding sites to support shadow detection, but that is out of scope for this post

and that is [`lc`](https://github.com/hljeong/cc/tree/7bcee9ff3ab01b82aed5b5798793f7dbdf7cd4f9/lambda-calculus) in its current state: a lambda calculus reducer that parses and evaluates. there is plenty more to explore -- church encodings, type systems, etc. alas, it is past midnight and i hear youtube's calling. until next time
