---
title: "church encoding"
date: 2026-04-04T21:51:15-07:00
tags: ["computer science"]
---

check this out: $\mathrm{math}$

anyways, let us explore church encoding today. last time we talked about lambda calculus ($\lambda$-calculus) and implemented a beta-reduction ($\beta$-reduction) engine. we claimed lambda calculus to be a "model of computation" but so far it has done nothing apart from being a brain stimulant as we look for the next beta-redex to reduce. church encoding is a way to assign familiar names to some of these unfamiliar lambda terms, named after alonzo church. beware that i am only taking the very basic definitions and extrapolating the rest of the mathematics mostly on my own, so they may deviate from the "canonical" church encodings

we begin with the church booleans, noting that $\lambda xy.M$ is an abbreviation for $\lambda x.\lambda y.M$:
$$
  \begin{align*}
    \mathrm{true} &:= \lambda xy.x \\\\
    \mathrm{false} &:= \lambda xy.y
  \end{align*}
$$

it may seem arbitrary to call these lambda terms booleans, but let us motivate these definitions by considering some function $\mathrm{if}$ that takes 3 inputs $p$, $x$, and $y$ and outputs $x$ if the predicate $p$ evaluates to $\mathrm{true}$ and outputs $y$ otherwise (read: if $p$ then $x$ else $y$). you may find that such a function is as simple as
$$
  \mathrm{if} := \lambda pxy.p\ x\ y
$$

indeed, let us try plugging in $\mathrm{true}$ and $\mathrm{false}$:
$$
  \begin{align*}
    \mathrm{if}\ \mathrm{true}\ x\ y  &\equiv (\lambda pxy.p\ x\ y)\ \mathrm{true}\ x\ y \\\\
                                      &=\_\beta \mathrm{true}\ x\ y \\\\
                                      &\equiv (\lambda xy.x )\ x\ y \\\\
                                      &=\_\beta x \\\\
    \mathrm{if}\ \mathrm{false}\ x\ y &\equiv (\lambda pxy.p\ x\ y)\ \mathrm{false}\ x\ y \\\\
                                      &=\_\beta \mathrm{false}\ x\ y \\\\
                                      &\equiv (\lambda xy.y)\ x\ y \\\\
                                      &=\_\beta y
  \end{align*}
$$
<!-- wtf why do i need to escape the underscores for it to render -->

in fact we see that the function $\mathrm{if}$ is not needed at all since $\mathrm{true}$ and $\mathrm{false}$ are already selectors themselves. similarly, if we had some more complicated predicate that evaluates to $\mathrm{true}$ or $\mathrm{false}$, we can simply write $(p\ x\ y)$ to convey "if $p$ then $x$ else $y$". for instance, the expression $\mathrm{not}\ p$ should return "if $p$ then $\mathrm{false}$ else $\mathrm{true}$". then we can write its definition as:
$$
  \mathrm{not} := \lambda p.p\ \mathrm{false}\ \mathrm{true} = \lambda pxy.p\ y\ x
$$

and the other boolean operators follow:
$$
  \begin{alignat*}{}
    \mathrm{and} &:= \lambda pq.p\ q\ \mathrm{false} &&= \lambda pqxy.p\ (q\ x\ y)\ y \\\\
    \mathrm{or}  &:= \lambda pq.p\ \mathrm{true}\ q &&= \lambda pqxy.p\ x\ (q\ x\ y) \\\\
    \mathrm{xor} &:= \lambda pq.p\ (\mathrm{not}\ q)\ q &&= \lambda pqxy.p\ (q\ y\ x)\ (q\ x\ y) \\\\
    \mathrm{nand} &:= \lambda pq.p\ (\mathrm{not}\ q)\ \mathrm{true} &&= \lambda pqxy.p\ (q\ y\ x)\ x \\\\
    \mathrm{nor} &:= \lambda pq.p\ \mathrm{false}\ (\mathrm{not}\ q)\ &&= \lambda pqxy.p\ y\ (q\ y\ x) \\\\
    \mathrm{nxor} &:= \lambda pq.p\ q\ (\mathrm{not}\ q) &&= \lambda pqxy.p\ (q\ x\ y)\ (q\ y\ x)
  \end{alignat*}
$$

church numerals encode natural numbers as repeated applications of a function, which feels quite natural to me:
$$
  \begin{alignat*}{}
    \mathrm{0} &:= \lambda fx.x \\\\
    \mathrm{1} &:= \lambda fx.f\ x \\\\
    \mathrm{2} &:= \lambda fx.f\ (f\ x) \\\\
               &\ \ \ \vdots \\\\
    \mathrm{n} &:= \lambda fx.f^{\circ n}\ x \\\\
  \end{alignat*}
$$

with this definition, applying a function $f$ to $x$ $n$ times can be written as $(n\ f\ x)$. however, to do any mathematics we would need operators. a natural starting point is the successor function, which injects an extra fold of composition of $f$:
$$
  \mathrm{succ} := \lambda nfx. n\ f\ (f\ x)
$$

adding $n$ to $m$ is simply applying $\mathrm{succ}$ to $n$ $m$ times:
$$
  \+ := \lambda nm.m\ \mathrm{succ}\ n
$$

similar story for multiplication:
$$
  \times := \lambda nm.m\ (+\ n)\ 0
$$
in words: "apply $+\ \mathrm{n}$ to $\mathrm{0}$ $\mathrm{m}$ times"

and exponentiation:
$$
  \mathrm{exp} := \lambda nm. m\ (\times\ n)\ 1
$$

the predecessor function is not nearly as straight-forward to come up with, so we take a detour to first try and define $\mathrm{eq0}$, which returns $\mathrm{true}$ when given input $\mathrm{0}$ and $\mathrm{false}$ for every other church numeral. considering church numerals apply some function $f$ to some $x$, we need to find $f$ and $x$ such that:
1. $\mathrm{0}\ f\ x =_\beta \mathrm{true}$
2. $\mathrm{n}\ f\ x =_\beta \mathrm{false}$ for all $\mathrm{n} > \mathrm{0}$

(1) gives us $x = \mathrm{true}$, and we shall define $f := \lambda x.\mathrm{false}$ to satisfy (2). then we have
$$
  \mathrm{eq0} := \lambda n.n\ (\lambda x.\mathrm{false})\ \mathrm{true}
$$

the key insight was choosing an $f$ that stays constant regardless of input. we will use a similar trick for $\mathrm{pred}$, but we will need to carry a bit more information. while there are no negative numbers in church numerals, we shall define $(\mathrm{pred}\ \mathrm{0})$ to return $\mathrm{0}$ which is the only church numeral that makes sense to return. let us try our previous strategy again. we need to find $f$ and $x$ such that:
1. $\mathrm{0}\ f\ x =_\beta \mathrm{0}$
2. $\mathrm{1}\ f\ x =_\beta \mathrm{0}$
3. $\mathrm{n}\ f\ (f\ x) =_\beta \mathrm{n}$ for all $\mathrm{n} > \mathrm{0}$

this is clearly impossible, since (1) gives us $x = \mathrm{0}$ and (2) gives us $f\ \mathrm{0} = \mathrm{0}$, meaning $\mathrm{n}\ f\ x = \mathrm{0}$ for all $\mathrm{n}$, contradicting (3). we need to somehow remember that we have applied $f$ once already. we shall do this by carrying a boolean denoting whether we have applied $f$ for the first time, but we first need to define a structure that allows us to carry two things at once -- a pair:
$$
  \begin{alignat*}{}
    \mathrm{pair}   &:= \lambda xyk.k\ x\ y \\\\
    \mathrm{first}  &:= \lambda p.p\ (\lambda xy.x) \\\\
    \mathrm{second} &:= \lambda p.p\ (\lambda xy.y)
  \end{alignat*}
$$

the function $\mathrm{pair}$ acts as a constructor: $\mathrm{pair}\ x\ y$ "stores" $x$ and $y$ and passes them to a continuation $k$ that can decide on what to do with them. in the case of $\mathrm{first}$, it takes the pair and returns the first element. the selector is technically the church boolean $\mathrm{true}$, but i find it awkward to think of it that way

now that we have a way of encoding a pair of values in a single term, we would now like to find $f$ such that:
- $f\ (\mathrm{pair}\ \mathrm{0}\ \mathrm{false}) = \mathrm{pair}\ \mathrm{0}\ \mathrm{true}$ -- don't increment, only set flag. this creates the lag we need
- $f\ (\mathrm{pair}\ x\ \mathrm{true}) = \mathrm{pair}\ (\mathrm{succ}\ x)\ \mathrm{true}$ -- increment the value if the flag has been set

we can easily write this function out:
$$
    f := \lambda p.p\ (\lambda xs.s\ (\mathrm{pair}\ (\mathrm{succ}\ x)\ \mathrm{true})\ (\mathrm{pair}\ 0\ \mathrm{true}))
$$

let us break this down. $x$ represents the first item in the pair, which is the return value. $s$ represents the second item in the pair, which is the flag that denotes whether we have applied $f$ at least once. we select one of the two branches based on the truthiness of $s$:
- if $s$, we have already applied $f$, so increment the value and return $(\mathrm{pair}\ (\mathrm{succ}\ x)\ \mathrm{true})$
- otherwise we are applying $f$ for the first time, return $(\mathrm{pair}\ \mathrm{0}\ \mathrm{true})$

using $\mathrm{first}$ to extract the value and discard the flag, we have:
1. $\mathrm{first}\ (\mathrm{0}\ f\ (\mathrm{pair}\ \mathrm{0}\ \mathrm{false})) = \mathrm{first}\ (\mathrm{pair}\ \mathrm{0}\ \mathrm{false}) = \mathrm{0}$
2. $\mathrm{first}\ (\mathrm{1}\ f\ (\mathrm{pair}\ \mathrm{0}\ \mathrm{false})) = \mathrm{first}\ (\mathrm{pair}\ \mathrm{0}\ \mathrm{true}) = \mathrm{0}$
3. $\mathrm{first}\ (\mathrm{n}\ f\ (f\ (\mathrm{pair}\ \mathrm{0}\ \mathrm{false}))) = \mathrm{first}\ (\mathrm{n}\ f\ (\mathrm{pair}\ \mathrm{0}\ \mathrm{true})) = \mathrm{first}\ (\mathrm{pair}\ \mathrm{n}\ \mathrm{true}) = \mathrm{n}$ for all $\mathrm{n} > \mathrm{0}$ by induction

at last, we can define:
$$
  \begin{alignat*}{}
    \mathrm{f} &:= \lambda p.p\ (\lambda xs.s\ (\mathrm{pair}\ (\mathrm{succ}\ x)\ \mathrm{true})\ (\mathrm{pair}\ \mathrm{0}\ \mathrm{true})) \\\\
    \mathrm{z} &:= \mathrm{pair}\ \mathrm{0}\ \mathrm{false} \\\\
    \mathrm{pred} &:= \lambda n. \mathrm{first}\ (n\ \mathrm{f}\ \mathrm{z})
  \end{alignat*}
$$

let us use the reduction engine we implemented last time to check our work. due to the complexity of the lambda term, let us write a quick python script to generate it:
```py
true = r"(\x.\y.x)"
false = r"(\x.\y.y)"
zero = r"(\f.\x.x)"
succ = r"(\n.\f.\x.n f (f x))"
numeral = lambda n: f"({succ} {numeral(n - 1)})" if n else zero
pair = r"(\x.\y.\k.k x y)"
first = r"(\p.p (\x.\y.x))"
f = f"(\\p.p (\\x.\\s.s ({pair} ({succ} x) {true}) ({pair} {zero} {true})))"
z = f"({pair} {zero} {false})"
pred = f"(\\n.{first} (n {f} {z}))"
print(f"{pred} {zero}")
print(f"{pred} {numeral(1)}")
print(f"{pred} {numeral(2)}")
print(f"{pred} {numeral(3)}")
```

behold the beauty:
```
# pred 0
(\n.(\p.p (\x.\y.x)) (n (\p.p (\x.\s.s ((\x.\y.\k.k x y) ((\n.\f.\x.n f (f x)) x) (\x.\y.x)) ((\x.\y.\k.k x y) (\f.\x.x) (\x.\y.x)))) ((\x.\y.\k.k x y) (\f.\x.x) (\x.\y.y)))) (\f.\x.x)
1: ((\p.(p (\x.(\y.x)))) (((\f.(\x.x)) (\p.(p (\x.(\s.((s (((\x.(\y.(\k.((k x) y)))) ((\n.(\f.(\x.((n f) (f x))))) x)) (\x.(\y.x)))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.x))))))))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.y)))))
2: ((((\f.(\x.x)) (\p.(p (\x.(\s.((s (((\x.(\y.(\k.((k x) y)))) ((\n.(\f.(\x.((n f) (f x))))) x)) (\x.(\y.x)))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.x))))))))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.y)))) (\x.(\y.x)))
3: (((\x.x) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.y)))) (\x.(\y.x)))
4: ((((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.y))) (\x.(\y.x)))
5: (((\y.(\k.((k (\f.(\x.x))) y))) (\x.(\y.y))) (\x.(\y.x)))
6: ((\k.((k (\f.(\x.x))) (\x.(\y.y)))) (\x.(\y.x)))
7: (((\x.(\y.x)) (\f.(\x.x))) (\x.(\y.y)))
8: ((\y.(\f.(\x.x))) (\x.(\y.y)))
9: (\f.(\x.x))
(\f.(\x.x))  # = 0

# pred 1
(\n.(\p.p (\x.\y.x)) (n (\p.p (\x.\s.s ((\x.\y.\k.k x y) ((\n.\f.\x.n f (f x)) x) (\x.\y.x)) ((\x.\y.\k.k x y) (\f.\x.x) (\x.\y.x)))) ((\x.\y.\k.k x y) (\f.\x.x) (\x.\y.y)))) ((\n.\f.\x.n f (f x)) (\f.\x.x))
1: ((\p.(p (\x.(\y.x)))) ((((\n.(\f.(\x.((n f) (f x))))) (\f.(\x.x))) (\p.(p (\x.(\s.((s (((\x.(\y.(\k.((k x) y)))) ((\n.(\f.(\x.((n f) (f x))))) x)) (\x.(\y.x)))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.x))))))))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.y)))))
2: (((((\n.(\f.(\x.((n f) (f x))))) (\f.(\x.x))) (\p.(p (\x.(\s.((s (((\x.(\y.(\k.((k x) y)))) ((\n.(\f.(\x.((n f) (f x))))) x)) (\x.(\y.x)))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.x))))))))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.y)))) (\x.(\y.x)))
3: ((((\f.(\x.(((\f.(\x.x)) f) (f x)))) (\p.(p (\x.(\s.((s (((\x.(\y.(\k.((k x) y)))) ((\n.(\f.(\x.((n f) (f x))))) x)) (\x.(\y.x)))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.x))))))))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.y)))) (\x.(\y.x)))
4: (((\x.(((\f.(\x.x)) (\p.(p (\x.(\s.((s (((\x.(\y.(\k.((k x) y)))) ((\n.(\f.(\x.((n f) (f x))))) x)) (\x.(\y.x)))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.x))))))))) ((\p.(p (\x.(\s.((s (((\x.(\y.(\k.((k x) y)))) ((\n.(\f.(\x.((n f) (f x))))) x)) (\x.(\y.x)))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.x)))))))) x))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.y)))) (\x.(\y.x)))
5: ((((\f.(\x.x)) (\p.(p (\x.(\s.((s (((\x.(\y.(\k.((k x) y)))) ((\n.(\f.(\x.((n f) (f x))))) x)) (\x.(\y.x)))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.x))))))))) ((\p.(p (\x.(\s.((s (((\x.(\y.(\k.((k x) y)))) ((\n.(\f.(\x.((n f) (f x))))) x)) (\x.(\y.x)))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.x)))))))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.y))))) (\x.(\y.x)))
6: (((\x.x) ((\p.(p (\x.(\s.((s (((\x.(\y.(\k.((k x) y)))) ((\n.(\f.(\x.((n f) (f x))))) x)) (\x.(\y.x)))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.x)))))))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.y))))) (\x.(\y.x)))
7: (((\p.(p (\x.(\s.((s (((\x.(\y.(\k.((k x) y)))) ((\n.(\f.(\x.((n f) (f x))))) x)) (\x.(\y.x)))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.x)))))))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.y)))) (\x.(\y.x)))
8: (((((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.y))) (\x.(\s.((s (((\x.(\y.(\k.((k x) y)))) ((\n.(\f.(\x.((n f) (f x))))) x)) (\x.(\y.x)))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.x))))))) (\x.(\y.x)))
9: ((((\y.(\k.((k (\f.(\x.x))) y))) (\x.(\y.y))) (\x.(\s.((s (((\x.(\y.(\k.((k x) y)))) ((\n.(\f.(\x.((n f) (f x))))) x)) (\x.(\y.x)))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.x))))))) (\x.(\y.x)))
10: (((\k.((k (\f.(\x.x))) (\x.(\y.y)))) (\x.(\s.((s (((\x.(\y.(\k.((k x) y)))) ((\n.(\f.(\x.((n f) (f x))))) x)) (\x.(\y.x)))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.x))))))) (\x.(\y.x)))
11: ((((\x.(\s.((s (((\x.(\y.(\k.((k x) y)))) ((\n.(\f.(\x.((n f) (f x))))) x)) (\x.(\y.x)))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.x)))))) (\f.(\x.x))) (\x.(\y.y))) (\x.(\y.x)))
12: (((\s.((s (((\x.(\y.(\k.((k x) y)))) ((\n.(\f.(\x.((n f) (f x))))) (\f.(\x.x)))) (\x.(\y.x)))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.x))))) (\x.(\y.y))) (\x.(\y.x)))
13: ((((\x.(\y.y)) (((\x.(\y.(\k.((k x) y)))) ((\n.(\f.(\x.((n f) (f x))))) (\f.(\x.x)))) (\x.(\y.x)))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.x)))) (\x.(\y.x)))
14: (((\y.y) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.x)))) (\x.(\y.x)))
15: ((((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.x))) (\x.(\y.x)))
16: (((\y.(\k.((k (\f.(\x.x))) y))) (\x.(\y.x))) (\x.(\y.x)))
17: ((\k.((k (\f.(\x.x))) (\x.(\y.x)))) (\x.(\y.x)))
18: (((\x.(\y.x)) (\f.(\x.x))) (\x.(\y.x)))
19: ((\y.(\f.(\x.x))) (\x.(\y.x)))
20: (\f.(\x.x))
(\f.(\x.x))  # = 0

# pred 2
(\n.(\p.p (\x.\y.x)) (n (\p.p (\x.\s.s ((\x.\y.\k.k x y) ((\n.\f.\x.n f (f x)) x) (\x.\y.x)) ((\x.\y.\k.k x y) (\f.\x.x) (\x.\y.x)))) ((\x.\y.\k.k x y) (\f.\x.x) (\x.\y.y)))) ((\n.\f.\x.n f (f x)) ((\n.\f.\x.n f (f x)) (\f.\x.x)))
1: ((\p.(p (\x.(\y.x)))) ((((\n.(\f.(\x.((n f) (f x))))) ((\n.(\f.(\x.((n f) (f x))))) (\f.(\x.x)))) (\p.(p (\x.(\s.((s (((\x.(\y.(\k.((k x) y)))) ((\n.(\f.(\x.((n f) (f x))))) x)) (\x.(\y.x)))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.x))))))))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.y)))))
2: (((((\n.(\f.(\x.((n f) (f x))))) ((\n.(\f.(\x.((n f) (f x))))) (\f.(\x.x)))) (\p.(p (\x.(\s.((s (((\x.(\y.(\k.((k x) y)))) ((\n.(\f.(\x.((n f) (f x))))) x)) (\x.(\y.x)))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.x))))))))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.y)))) (\x.(\y.x)))
3: ((((\f.(\x.((((\n.(\f.(\x.((n f) (f x))))) (\f.(\x.x))) f) (f x)))) (\p.(p (\x.(\s.((s (((\x.(\y.(\k.((k x) y)))) ((\n.(\f.(\x.((n f) (f x))))) x)) (\x.(\y.x)))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.x))))))))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.y)))) (\x.(\y.x)))
4: (((\x.((((\n.(\f.(\x.((n f) (f x))))) (\f.(\x.x))) (\p.(p (\x.(\s.((s (((\x.(\y.(\k.((k x) y)))) ((\n.(\f.(\x.((n f) (f x))))) x)) (\x.(\y.x)))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.x))))))))) ((\p.(p (\x.(\s.((s (((\x.(\y.(\k.((k x) y)))) ((\n.(\f.(\x.((n f) (f x))))) x)) (\x.(\y.x)))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.x)))))))) x))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.y)))) (\x.(\y.x)))
5: (((((\n.(\f.(\x.((n f) (f x))))) (\f.(\x.x))) (\p.(p (\x.(\s.((s (((\x.(\y.(\k.((k x) y)))) ((\n.(\f.(\x.((n f) (f x))))) x)) (\x.(\y.x)))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.x))))))))) ((\p.(p (\x.(\s.((s (((\x.(\y.(\k.((k x) y)))) ((\n.(\f.(\x.((n f) (f x))))) x)) (\x.(\y.x)))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.x)))))))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.y))))) (\x.(\y.x)))
6: ((((\f.(\x.(((\f.(\x.x)) f) (f x)))) (\p.(p (\x.(\s.((s (((\x.(\y.(\k.((k x) y)))) ((\n.(\f.(\x.((n f) (f x))))) x)) (\x.(\y.x)))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.x))))))))) ((\p.(p (\x.(\s.((s (((\x.(\y.(\k.((k x) y)))) ((\n.(\f.(\x.((n f) (f x))))) x)) (\x.(\y.x)))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.x)))))))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.y))))) (\x.(\y.x)))
7: (((\x.(((\f.(\x.x)) (\p.(p (\x.(\s.((s (((\x.(\y.(\k.((k x) y)))) ((\n.(\f.(\x.((n f) (f x))))) x)) (\x.(\y.x)))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.x))))))))) ((\p.(p (\x.(\s.((s (((\x.(\y.(\k.((k x) y)))) ((\n.(\f.(\x.((n f) (f x))))) x)) (\x.(\y.x)))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.x)))))))) x))) ((\p.(p (\x.(\s.((s (((\x.(\y.(\k.((k x) y)))) ((\n.(\f.(\x.((n f) (f x))))) x)) (\x.(\y.x)))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.x)))))))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.y))))) (\x.(\y.x)))
8: ((((\f.(\x.x)) (\p.(p (\x.(\s.((s (((\x.(\y.(\k.((k x) y)))) ((\n.(\f.(\x.((n f) (f x))))) x)) (\x.(\y.x)))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.x))))))))) ((\p.(p (\x.(\s.((s (((\x.(\y.(\k.((k x) y)))) ((\n.(\f.(\x.((n f) (f x))))) x)) (\x.(\y.x)))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.x)))))))) ((\p.(p (\x.(\s.((s (((\x.(\y.(\k.((k x) y)))) ((\n.(\f.(\x.((n f) (f x))))) x)) (\x.(\y.x)))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.x)))))))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.y)))))) (\x.(\y.x)))
9: (((\x.x) ((\p.(p (\x.(\s.((s (((\x.(\y.(\k.((k x) y)))) ((\n.(\f.(\x.((n f) (f x))))) x)) (\x.(\y.x)))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.x)))))))) ((\p.(p (\x.(\s.((s (((\x.(\y.(\k.((k x) y)))) ((\n.(\f.(\x.((n f) (f x))))) x)) (\x.(\y.x)))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.x)))))))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.y)))))) (\x.(\y.x)))
10: (((\p.(p (\x.(\s.((s (((\x.(\y.(\k.((k x) y)))) ((\n.(\f.(\x.((n f) (f x))))) x)) (\x.(\y.x)))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.x)))))))) ((\p.(p (\x.(\s.((s (((\x.(\y.(\k.((k x) y)))) ((\n.(\f.(\x.((n f) (f x))))) x)) (\x.(\y.x)))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.x)))))))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.y))))) (\x.(\y.x)))
11: ((((\p.(p (\x.(\s.((s (((\x.(\y.(\k.((k x) y)))) ((\n.(\f.(\x.((n f) (f x))))) x)) (\x.(\y.x)))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.x)))))))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.y)))) (\x.(\s.((s (((\x.(\y.(\k.((k x) y)))) ((\n.(\f.(\x.((n f) (f x))))) x)) (\x.(\y.x)))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.x))))))) (\x.(\y.x)))
12: ((((((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.y))) (\x.(\s.((s (((\x.(\y.(\k.((k x) y)))) ((\n.(\f.(\x.((n f) (f x))))) x)) (\x.(\y.x)))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.x))))))) (\x.(\s.((s (((\x.(\y.(\k.((k x) y)))) ((\n.(\f.(\x.((n f) (f x))))) x)) (\x.(\y.x)))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.x))))))) (\x.(\y.x)))
13: (((((\y.(\k.((k (\f.(\x.x))) y))) (\x.(\y.y))) (\x.(\s.((s (((\x.(\y.(\k.((k x) y)))) ((\n.(\f.(\x.((n f) (f x))))) x)) (\x.(\y.x)))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.x))))))) (\x.(\s.((s (((\x.(\y.(\k.((k x) y)))) ((\n.(\f.(\x.((n f) (f x))))) x)) (\x.(\y.x)))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.x))))))) (\x.(\y.x)))
14: ((((\k.((k (\f.(\x.x))) (\x.(\y.y)))) (\x.(\s.((s (((\x.(\y.(\k.((k x) y)))) ((\n.(\f.(\x.((n f) (f x))))) x)) (\x.(\y.x)))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.x))))))) (\x.(\s.((s (((\x.(\y.(\k.((k x) y)))) ((\n.(\f.(\x.((n f) (f x))))) x)) (\x.(\y.x)))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.x))))))) (\x.(\y.x)))
15: (((((\x.(\s.((s (((\x.(\y.(\k.((k x) y)))) ((\n.(\f.(\x.((n f) (f x))))) x)) (\x.(\y.x)))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.x)))))) (\f.(\x.x))) (\x.(\y.y))) (\x.(\s.((s (((\x.(\y.(\k.((k x) y)))) ((\n.(\f.(\x.((n f) (f x))))) x)) (\x.(\y.x)))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.x))))))) (\x.(\y.x)))
16: ((((\s.((s (((\x.(\y.(\k.((k x) y)))) ((\n.(\f.(\x.((n f) (f x))))) (\f.(\x.x)))) (\x.(\y.x)))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.x))))) (\x.(\y.y))) (\x.(\s.((s (((\x.(\y.(\k.((k x) y)))) ((\n.(\f.(\x.((n f) (f x))))) x)) (\x.(\y.x)))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.x))))))) (\x.(\y.x)))
17: (((((\x.(\y.y)) (((\x.(\y.(\k.((k x) y)))) ((\n.(\f.(\x.((n f) (f x))))) (\f.(\x.x)))) (\x.(\y.x)))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.x)))) (\x.(\s.((s (((\x.(\y.(\k.((k x) y)))) ((\n.(\f.(\x.((n f) (f x))))) x)) (\x.(\y.x)))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.x))))))) (\x.(\y.x)))
18: ((((\y.y) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.x)))) (\x.(\s.((s (((\x.(\y.(\k.((k x) y)))) ((\n.(\f.(\x.((n f) (f x))))) x)) (\x.(\y.x)))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.x))))))) (\x.(\y.x)))
19: (((((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.x))) (\x.(\s.((s (((\x.(\y.(\k.((k x) y)))) ((\n.(\f.(\x.((n f) (f x))))) x)) (\x.(\y.x)))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.x))))))) (\x.(\y.x)))
20: ((((\y.(\k.((k (\f.(\x.x))) y))) (\x.(\y.x))) (\x.(\s.((s (((\x.(\y.(\k.((k x) y)))) ((\n.(\f.(\x.((n f) (f x))))) x)) (\x.(\y.x)))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.x))))))) (\x.(\y.x)))
21: (((\k.((k (\f.(\x.x))) (\x.(\y.x)))) (\x.(\s.((s (((\x.(\y.(\k.((k x) y)))) ((\n.(\f.(\x.((n f) (f x))))) x)) (\x.(\y.x)))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.x))))))) (\x.(\y.x)))
22: ((((\x.(\s.((s (((\x.(\y.(\k.((k x) y)))) ((\n.(\f.(\x.((n f) (f x))))) x)) (\x.(\y.x)))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.x)))))) (\f.(\x.x))) (\x.(\y.x))) (\x.(\y.x)))
23: (((\s.((s (((\x.(\y.(\k.((k x) y)))) ((\n.(\f.(\x.((n f) (f x))))) (\f.(\x.x)))) (\x.(\y.x)))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.x))))) (\x.(\y.x))) (\x.(\y.x)))
24: ((((\x.(\y.x)) (((\x.(\y.(\k.((k x) y)))) ((\n.(\f.(\x.((n f) (f x))))) (\f.(\x.x)))) (\x.(\y.x)))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.x)))) (\x.(\y.x)))
25: (((\y.(((\x.(\y.(\k.((k x) y)))) ((\n.(\f.(\x.((n f) (f x))))) (\f.(\x.x)))) (\x.(\y.x)))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.x)))) (\x.(\y.x)))
26: ((((\x.(\y.(\k.((k x) y)))) ((\n.(\f.(\x.((n f) (f x))))) (\f.(\x.x)))) (\x.(\y.x))) (\x.(\y.x)))
27: (((\y.(\k.((k ((\n.(\f.(\x.((n f) (f x))))) (\f.(\x.x)))) y))) (\x.(\y.x))) (\x.(\y.x)))
28: ((\k.((k ((\n.(\f.(\x.((n f) (f x))))) (\f.(\x.x)))) (\x.(\y.x)))) (\x.(\y.x)))
29: (((\x.(\y.x)) ((\n.(\f.(\x.((n f) (f x))))) (\f.(\x.x)))) (\x.(\y.x)))
30: ((\y.((\n.(\f.(\x.((n f) (f x))))) (\f.(\x.x)))) (\x.(\y.x)))
31: ((\n.(\f.(\x.((n f) (f x))))) (\f.(\x.x)))
32: (\f.(\x.(((\f.(\x.x)) f) (f x))))
33: (\f.(\x.((\x.x) (f x))))
34: (\f.(\x.(f x)))
(\f.(\x.(f x)))  # = 1

# pred 3
(\n.(\p.p (\x.\y.x)) (n (\p.p (\x.\s.s ((\x.\y.\k.k x y) ((\n.\f.\x.n f (f x)) x) (\x.\y.x)) ((\x.\y.\k.k x y) (\f.\x.x) (\x.\y.x)))) ((\x.\y.\k.k x y) (\f.\x.x) (\x.\y.y)))) ((\n.\f.\x.n f (f x)) ((\n.\f.\x.n f (f x)) ((\n.\f.\x.n f (f x)) (\f.\x.x))))
1: ((\p.(p (\x.(\y.x)))) ((((\n.(\f.(\x.((n f) (f x))))) ((\n.(\f.(\x.((n f) (f x))))) ((\n.(\f.(\x.((n f) (f x))))) (\f.(\x.x))))) (\p.(p (\x.(\s.((s (((\x.(\y.(\k.((k x) y)))) ((\n.(\f.(\x.((n f) (f x))))) x)) (\x.(\y.x)))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.x))))))))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.y)))))
2: (((((\n.(\f.(\x.((n f) (f x))))) ((\n.(\f.(\x.((n f) (f x))))) ((\n.(\f.(\x.((n f) (f x))))) (\f.(\x.x))))) (\p.(p (\x.(\s.((s (((\x.(\y.(\k.((k x) y)))) ((\n.(\f.(\x.((n f) (f x))))) x)) (\x.(\y.x)))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.x))))))))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.y)))) (\x.(\y.x)))
3: ((((\f.(\x.((((\n.(\f.(\x.((n f) (f x))))) ((\n.(\f.(\x.((n f) (f x))))) (\f.(\x.x)))) f) (f x)))) (\p.(p (\x.(\s.((s (((\x.(\y.(\k.((k x) y)))) ((\n.(\f.(\x.((n f) (f x))))) x)) (\x.(\y.x)))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.x))))))))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.y)))) (\x.(\y.x)))
4: (((\x.((((\n.(\f.(\x.((n f) (f x))))) ((\n.(\f.(\x.((n f) (f x))))) (\f.(\x.x)))) (\p.(p (\x.(\s.((s (((\x.(\y.(\k.((k x) y)))) ((\n.(\f.(\x.((n f) (f x))))) x)) (\x.(\y.x)))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.x))))))))) ((\p.(p (\x.(\s.((s (((\x.(\y.(\k.((k x) y)))) ((\n.(\f.(\x.((n f) (f x))))) x)) (\x.(\y.x)))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.x)))))))) x))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.y)))) (\x.(\y.x)))
5: (((((\n.(\f.(\x.((n f) (f x))))) ((\n.(\f.(\x.((n f) (f x))))) (\f.(\x.x)))) (\p.(p (\x.(\s.((s (((\x.(\y.(\k.((k x) y)))) ((\n.(\f.(\x.((n f) (f x))))) x)) (\x.(\y.x)))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.x))))))))) ((\p.(p (\x.(\s.((s (((\x.(\y.(\k.((k x) y)))) ((\n.(\f.(\x.((n f) (f x))))) x)) (\x.(\y.x)))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.x)))))))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.y))))) (\x.(\y.x)))
6: ((((\f.(\x.((((\n.(\f.(\x.((n f) (f x))))) (\f.(\x.x))) f) (f x)))) (\p.(p (\x.(\s.((s (((\x.(\y.(\k.((k x) y)))) ((\n.(\f.(\x.((n f) (f x))))) x)) (\x.(\y.x)))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.x))))))))) ((\p.(p (\x.(\s.((s (((\x.(\y.(\k.((k x) y)))) ((\n.(\f.(\x.((n f) (f x))))) x)) (\x.(\y.x)))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.x)))))))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.y))))) (\x.(\y.x)))
7: (((\x.((((\n.(\f.(\x.((n f) (f x))))) (\f.(\x.x))) (\p.(p (\x.(\s.((s (((\x.(\y.(\k.((k x) y)))) ((\n.(\f.(\x.((n f) (f x))))) x)) (\x.(\y.x)))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.x))))))))) ((\p.(p (\x.(\s.((s (((\x.(\y.(\k.((k x) y)))) ((\n.(\f.(\x.((n f) (f x))))) x)) (\x.(\y.x)))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.x)))))))) x))) ((\p.(p (\x.(\s.((s (((\x.(\y.(\k.((k x) y)))) ((\n.(\f.(\x.((n f) (f x))))) x)) (\x.(\y.x)))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.x)))))))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.y))))) (\x.(\y.x)))
8: (((((\n.(\f.(\x.((n f) (f x))))) (\f.(\x.x))) (\p.(p (\x.(\s.((s (((\x.(\y.(\k.((k x) y)))) ((\n.(\f.(\x.((n f) (f x))))) x)) (\x.(\y.x)))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.x))))))))) ((\p.(p (\x.(\s.((s (((\x.(\y.(\k.((k x) y)))) ((\n.(\f.(\x.((n f) (f x))))) x)) (\x.(\y.x)))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.x)))))))) ((\p.(p (\x.(\s.((s (((\x.(\y.(\k.((k x) y)))) ((\n.(\f.(\x.((n f) (f x))))) x)) (\x.(\y.x)))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.x)))))))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.y)))))) (\x.(\y.x)))
9: ((((\f.(\x.(((\f.(\x.x)) f) (f x)))) (\p.(p (\x.(\s.((s (((\x.(\y.(\k.((k x) y)))) ((\n.(\f.(\x.((n f) (f x))))) x)) (\x.(\y.x)))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.x))))))))) ((\p.(p (\x.(\s.((s (((\x.(\y.(\k.((k x) y)))) ((\n.(\f.(\x.((n f) (f x))))) x)) (\x.(\y.x)))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.x)))))))) ((\p.(p (\x.(\s.((s (((\x.(\y.(\k.((k x) y)))) ((\n.(\f.(\x.((n f) (f x))))) x)) (\x.(\y.x)))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.x)))))))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.y)))))) (\x.(\y.x)))
10: (((\x.(((\f.(\x.x)) (\p.(p (\x.(\s.((s (((\x.(\y.(\k.((k x) y)))) ((\n.(\f.(\x.((n f) (f x))))) x)) (\x.(\y.x)))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.x))))))))) ((\p.(p (\x.(\s.((s (((\x.(\y.(\k.((k x) y)))) ((\n.(\f.(\x.((n f) (f x))))) x)) (\x.(\y.x)))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.x)))))))) x))) ((\p.(p (\x.(\s.((s (((\x.(\y.(\k.((k x) y)))) ((\n.(\f.(\x.((n f) (f x))))) x)) (\x.(\y.x)))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.x)))))))) ((\p.(p (\x.(\s.((s (((\x.(\y.(\k.((k x) y)))) ((\n.(\f.(\x.((n f) (f x))))) x)) (\x.(\y.x)))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.x)))))))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.y)))))) (\x.(\y.x)))
11: ((((\f.(\x.x)) (\p.(p (\x.(\s.((s (((\x.(\y.(\k.((k x) y)))) ((\n.(\f.(\x.((n f) (f x))))) x)) (\x.(\y.x)))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.x))))))))) ((\p.(p (\x.(\s.((s (((\x.(\y.(\k.((k x) y)))) ((\n.(\f.(\x.((n f) (f x))))) x)) (\x.(\y.x)))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.x)))))))) ((\p.(p (\x.(\s.((s (((\x.(\y.(\k.((k x) y)))) ((\n.(\f.(\x.((n f) (f x))))) x)) (\x.(\y.x)))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.x)))))))) ((\p.(p (\x.(\s.((s (((\x.(\y.(\k.((k x) y)))) ((\n.(\f.(\x.((n f) (f x))))) x)) (\x.(\y.x)))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.x)))))))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.y))))))) (\x.(\y.x)))
12: (((\x.x) ((\p.(p (\x.(\s.((s (((\x.(\y.(\k.((k x) y)))) ((\n.(\f.(\x.((n f) (f x))))) x)) (\x.(\y.x)))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.x)))))))) ((\p.(p (\x.(\s.((s (((\x.(\y.(\k.((k x) y)))) ((\n.(\f.(\x.((n f) (f x))))) x)) (\x.(\y.x)))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.x)))))))) ((\p.(p (\x.(\s.((s (((\x.(\y.(\k.((k x) y)))) ((\n.(\f.(\x.((n f) (f x))))) x)) (\x.(\y.x)))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.x)))))))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.y))))))) (\x.(\y.x)))
13: (((\p.(p (\x.(\s.((s (((\x.(\y.(\k.((k x) y)))) ((\n.(\f.(\x.((n f) (f x))))) x)) (\x.(\y.x)))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.x)))))))) ((\p.(p (\x.(\s.((s (((\x.(\y.(\k.((k x) y)))) ((\n.(\f.(\x.((n f) (f x))))) x)) (\x.(\y.x)))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.x)))))))) ((\p.(p (\x.(\s.((s (((\x.(\y.(\k.((k x) y)))) ((\n.(\f.(\x.((n f) (f x))))) x)) (\x.(\y.x)))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.x)))))))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.y)))))) (\x.(\y.x)))
14: ((((\p.(p (\x.(\s.((s (((\x.(\y.(\k.((k x) y)))) ((\n.(\f.(\x.((n f) (f x))))) x)) (\x.(\y.x)))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.x)))))))) ((\p.(p (\x.(\s.((s (((\x.(\y.(\k.((k x) y)))) ((\n.(\f.(\x.((n f) (f x))))) x)) (\x.(\y.x)))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.x)))))))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.y))))) (\x.(\s.((s (((\x.(\y.(\k.((k x) y)))) ((\n.(\f.(\x.((n f) (f x))))) x)) (\x.(\y.x)))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.x))))))) (\x.(\y.x)))
15: (((((\p.(p (\x.(\s.((s (((\x.(\y.(\k.((k x) y)))) ((\n.(\f.(\x.((n f) (f x))))) x)) (\x.(\y.x)))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.x)))))))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.y)))) (\x.(\s.((s (((\x.(\y.(\k.((k x) y)))) ((\n.(\f.(\x.((n f) (f x))))) x)) (\x.(\y.x)))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.x))))))) (\x.(\s.((s (((\x.(\y.(\k.((k x) y)))) ((\n.(\f.(\x.((n f) (f x))))) x)) (\x.(\y.x)))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.x))))))) (\x.(\y.x)))
16: (((((((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.y))) (\x.(\s.((s (((\x.(\y.(\k.((k x) y)))) ((\n.(\f.(\x.((n f) (f x))))) x)) (\x.(\y.x)))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.x))))))) (\x.(\s.((s (((\x.(\y.(\k.((k x) y)))) ((\n.(\f.(\x.((n f) (f x))))) x)) (\x.(\y.x)))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.x))))))) (\x.(\s.((s (((\x.(\y.(\k.((k x) y)))) ((\n.(\f.(\x.((n f) (f x))))) x)) (\x.(\y.x)))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.x))))))) (\x.(\y.x)))
17: ((((((\y.(\k.((k (\f.(\x.x))) y))) (\x.(\y.y))) (\x.(\s.((s (((\x.(\y.(\k.((k x) y)))) ((\n.(\f.(\x.((n f) (f x))))) x)) (\x.(\y.x)))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.x))))))) (\x.(\s.((s (((\x.(\y.(\k.((k x) y)))) ((\n.(\f.(\x.((n f) (f x))))) x)) (\x.(\y.x)))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.x))))))) (\x.(\s.((s (((\x.(\y.(\k.((k x) y)))) ((\n.(\f.(\x.((n f) (f x))))) x)) (\x.(\y.x)))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.x))))))) (\x.(\y.x)))
18: (((((\k.((k (\f.(\x.x))) (\x.(\y.y)))) (\x.(\s.((s (((\x.(\y.(\k.((k x) y)))) ((\n.(\f.(\x.((n f) (f x))))) x)) (\x.(\y.x)))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.x))))))) (\x.(\s.((s (((\x.(\y.(\k.((k x) y)))) ((\n.(\f.(\x.((n f) (f x))))) x)) (\x.(\y.x)))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.x))))))) (\x.(\s.((s (((\x.(\y.(\k.((k x) y)))) ((\n.(\f.(\x.((n f) (f x))))) x)) (\x.(\y.x)))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.x))))))) (\x.(\y.x)))
19: ((((((\x.(\s.((s (((\x.(\y.(\k.((k x) y)))) ((\n.(\f.(\x.((n f) (f x))))) x)) (\x.(\y.x)))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.x)))))) (\f.(\x.x))) (\x.(\y.y))) (\x.(\s.((s (((\x.(\y.(\k.((k x) y)))) ((\n.(\f.(\x.((n f) (f x))))) x)) (\x.(\y.x)))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.x))))))) (\x.(\s.((s (((\x.(\y.(\k.((k x) y)))) ((\n.(\f.(\x.((n f) (f x))))) x)) (\x.(\y.x)))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.x))))))) (\x.(\y.x)))
20: (((((\s.((s (((\x.(\y.(\k.((k x) y)))) ((\n.(\f.(\x.((n f) (f x))))) (\f.(\x.x)))) (\x.(\y.x)))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.x))))) (\x.(\y.y))) (\x.(\s.((s (((\x.(\y.(\k.((k x) y)))) ((\n.(\f.(\x.((n f) (f x))))) x)) (\x.(\y.x)))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.x))))))) (\x.(\s.((s (((\x.(\y.(\k.((k x) y)))) ((\n.(\f.(\x.((n f) (f x))))) x)) (\x.(\y.x)))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.x))))))) (\x.(\y.x)))
21: ((((((\x.(\y.y)) (((\x.(\y.(\k.((k x) y)))) ((\n.(\f.(\x.((n f) (f x))))) (\f.(\x.x)))) (\x.(\y.x)))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.x)))) (\x.(\s.((s (((\x.(\y.(\k.((k x) y)))) ((\n.(\f.(\x.((n f) (f x))))) x)) (\x.(\y.x)))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.x))))))) (\x.(\s.((s (((\x.(\y.(\k.((k x) y)))) ((\n.(\f.(\x.((n f) (f x))))) x)) (\x.(\y.x)))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.x))))))) (\x.(\y.x)))
22: (((((\y.y) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.x)))) (\x.(\s.((s (((\x.(\y.(\k.((k x) y)))) ((\n.(\f.(\x.((n f) (f x))))) x)) (\x.(\y.x)))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.x))))))) (\x.(\s.((s (((\x.(\y.(\k.((k x) y)))) ((\n.(\f.(\x.((n f) (f x))))) x)) (\x.(\y.x)))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.x))))))) (\x.(\y.x)))
23: ((((((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.x))) (\x.(\s.((s (((\x.(\y.(\k.((k x) y)))) ((\n.(\f.(\x.((n f) (f x))))) x)) (\x.(\y.x)))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.x))))))) (\x.(\s.((s (((\x.(\y.(\k.((k x) y)))) ((\n.(\f.(\x.((n f) (f x))))) x)) (\x.(\y.x)))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.x))))))) (\x.(\y.x)))
24: (((((\y.(\k.((k (\f.(\x.x))) y))) (\x.(\y.x))) (\x.(\s.((s (((\x.(\y.(\k.((k x) y)))) ((\n.(\f.(\x.((n f) (f x))))) x)) (\x.(\y.x)))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.x))))))) (\x.(\s.((s (((\x.(\y.(\k.((k x) y)))) ((\n.(\f.(\x.((n f) (f x))))) x)) (\x.(\y.x)))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.x))))))) (\x.(\y.x)))
25: ((((\k.((k (\f.(\x.x))) (\x.(\y.x)))) (\x.(\s.((s (((\x.(\y.(\k.((k x) y)))) ((\n.(\f.(\x.((n f) (f x))))) x)) (\x.(\y.x)))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.x))))))) (\x.(\s.((s (((\x.(\y.(\k.((k x) y)))) ((\n.(\f.(\x.((n f) (f x))))) x)) (\x.(\y.x)))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.x))))))) (\x.(\y.x)))
26: (((((\x.(\s.((s (((\x.(\y.(\k.((k x) y)))) ((\n.(\f.(\x.((n f) (f x))))) x)) (\x.(\y.x)))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.x)))))) (\f.(\x.x))) (\x.(\y.x))) (\x.(\s.((s (((\x.(\y.(\k.((k x) y)))) ((\n.(\f.(\x.((n f) (f x))))) x)) (\x.(\y.x)))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.x))))))) (\x.(\y.x)))
27: ((((\s.((s (((\x.(\y.(\k.((k x) y)))) ((\n.(\f.(\x.((n f) (f x))))) (\f.(\x.x)))) (\x.(\y.x)))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.x))))) (\x.(\y.x))) (\x.(\s.((s (((\x.(\y.(\k.((k x) y)))) ((\n.(\f.(\x.((n f) (f x))))) x)) (\x.(\y.x)))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.x))))))) (\x.(\y.x)))
28: (((((\x.(\y.x)) (((\x.(\y.(\k.((k x) y)))) ((\n.(\f.(\x.((n f) (f x))))) (\f.(\x.x)))) (\x.(\y.x)))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.x)))) (\x.(\s.((s (((\x.(\y.(\k.((k x) y)))) ((\n.(\f.(\x.((n f) (f x))))) x)) (\x.(\y.x)))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.x))))))) (\x.(\y.x)))
29: ((((\y.(((\x.(\y.(\k.((k x) y)))) ((\n.(\f.(\x.((n f) (f x))))) (\f.(\x.x)))) (\x.(\y.x)))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.x)))) (\x.(\s.((s (((\x.(\y.(\k.((k x) y)))) ((\n.(\f.(\x.((n f) (f x))))) x)) (\x.(\y.x)))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.x))))))) (\x.(\y.x)))
30: (((((\x.(\y.(\k.((k x) y)))) ((\n.(\f.(\x.((n f) (f x))))) (\f.(\x.x)))) (\x.(\y.x))) (\x.(\s.((s (((\x.(\y.(\k.((k x) y)))) ((\n.(\f.(\x.((n f) (f x))))) x)) (\x.(\y.x)))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.x))))))) (\x.(\y.x)))
31: ((((\y.(\k.((k ((\n.(\f.(\x.((n f) (f x))))) (\f.(\x.x)))) y))) (\x.(\y.x))) (\x.(\s.((s (((\x.(\y.(\k.((k x) y)))) ((\n.(\f.(\x.((n f) (f x))))) x)) (\x.(\y.x)))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.x))))))) (\x.(\y.x)))
32: (((\k.((k ((\n.(\f.(\x.((n f) (f x))))) (\f.(\x.x)))) (\x.(\y.x)))) (\x.(\s.((s (((\x.(\y.(\k.((k x) y)))) ((\n.(\f.(\x.((n f) (f x))))) x)) (\x.(\y.x)))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.x))))))) (\x.(\y.x)))
33: ((((\x.(\s.((s (((\x.(\y.(\k.((k x) y)))) ((\n.(\f.(\x.((n f) (f x))))) x)) (\x.(\y.x)))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.x)))))) ((\n.(\f.(\x.((n f) (f x))))) (\f.(\x.x)))) (\x.(\y.x))) (\x.(\y.x)))
34: (((\s.((s (((\x.(\y.(\k.((k x) y)))) ((\n.(\f.(\x.((n f) (f x))))) ((\n.(\f.(\x.((n f) (f x))))) (\f.(\x.x))))) (\x.(\y.x)))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.x))))) (\x.(\y.x))) (\x.(\y.x)))
35: ((((\x.(\y.x)) (((\x.(\y.(\k.((k x) y)))) ((\n.(\f.(\x.((n f) (f x))))) ((\n.(\f.(\x.((n f) (f x))))) (\f.(\x.x))))) (\x.(\y.x)))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.x)))) (\x.(\y.x)))
36: (((\y.(((\x.(\y.(\k.((k x) y)))) ((\n.(\f.(\x.((n f) (f x))))) ((\n.(\f.(\x.((n f) (f x))))) (\f.(\x.x))))) (\x.(\y.x)))) (((\x.(\y.(\k.((k x) y)))) (\f.(\x.x))) (\x.(\y.x)))) (\x.(\y.x)))
37: ((((\x.(\y.(\k.((k x) y)))) ((\n.(\f.(\x.((n f) (f x))))) ((\n.(\f.(\x.((n f) (f x))))) (\f.(\x.x))))) (\x.(\y.x))) (\x.(\y.x)))
38: (((\y.(\k.((k ((\n.(\f.(\x.((n f) (f x))))) ((\n.(\f.(\x.((n f) (f x))))) (\f.(\x.x))))) y))) (\x.(\y.x))) (\x.(\y.x)))
39: ((\k.((k ((\n.(\f.(\x.((n f) (f x))))) ((\n.(\f.(\x.((n f) (f x))))) (\f.(\x.x))))) (\x.(\y.x)))) (\x.(\y.x)))
40: (((\x.(\y.x)) ((\n.(\f.(\x.((n f) (f x))))) ((\n.(\f.(\x.((n f) (f x))))) (\f.(\x.x))))) (\x.(\y.x)))
41: ((\y.((\n.(\f.(\x.((n f) (f x))))) ((\n.(\f.(\x.((n f) (f x))))) (\f.(\x.x))))) (\x.(\y.x)))
42: ((\n.(\f.(\x.((n f) (f x))))) ((\n.(\f.(\x.((n f) (f x))))) (\f.(\x.x))))
43: (\f.(\x.((((\n.(\f.(\x.((n f) (f x))))) (\f.(\x.x))) f) (f x))))
44: (\f.(\x.(((\f.(\x.(((\f.(\x.x)) f) (f x)))) f) (f x))))
45: (\f.(\x.((\x.(((\f.(\x.x)) f) (f x))) (f x))))
46: (\f.(\x.(((\f.(\x.x)) f) (f (f x)))))
47: (\f.(\x.((\x.x) (f (f x)))))
48: (\f.(\x.(f (f x))))
(\f.(\x.(f (f x))))  # = 2
```

## epilogue

so i just had the thought of "why tf am i writing in this tone" -- this blog is going to reach approximately 3 people maximum, and i'm being pedagogical for _what_

inauthentic fk. next blog will be better i promise
