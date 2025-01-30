---
title: "c++ template-like invocation in python"
date: 2025-01-29T23:05:11-08:00
---

so the other day for whatever reason i really wanted to invoke python functions as if they were c++ template functions... like so:
```cpp
template <int x>
int add(int y) {
    return x + y;
}


printf("%d\n", add<5>(3));
```

if we ignore semantics, `add<5>(3)` seems to be a valid python expression -- roughly translating to `add.__lt__(5).__gt__(3)`

i quickly whipped up a prototype:
```py
class TemplateAdd:
    def __init__(self):
        self.x = None

    def __lt__(self, x):
        self.x = x
        return self

    def __gt__(self, y):
        return self.x + y


add = TemplateAdd()
print(add<5>(3))
```

but somehow it prints `True`... thanks, [chained comparisons](https://docs.python.org/3/reference/expressions.html#comparisons). the expression `add<5>(3)` evaluates to `add.__lt__(5) and 5 > 3`, and with no way to overload the `and` operator, the dream is dead unless i come to terms with the deranged syntax of `(add<5)>3` (doesnt `<5)>3` kind of look like a fish?)

i then settled for the sane option, which i probably shouldve gone with the first time around: `add[5](3)`. easy enough, when the language is on your side:
```py
class TemplateAdd:
    def __getitem__(self, x):
        return lambda y: x + y


add = TemplateAdd()
print(add[5](3))
```

but of course im not writing a new class every time i want a template function, luckily python decorators provide a syntax very similar to c++ templates. the desired syntax in a perfect world looks something like:
```py
@template[x]
def add(y):
    return x + y
```

referencing the undefined name `x` is an original sin, especially the one passed to the decorator, where there is no space for shenanigans -- alas, how many times have i wished to be able to customize `globals().__getitem__`

let us take a step back:
```py
@template("x")
def add(y, *, x):
    return x + y
```

this syntax has the approval of my lsp, and it even supports optionally passing in the template parameter `x` directly -- but i see that as a shortcoming, since thats not how c++ templates work
```py
class TemplateFunction:
    def __init__(self, func, argname):
        self.func = func
        self.argname = argname

    def __call__(self, *a, **kw):
        return self.func(*a, **kw)

    def __getitem__(self, argvalue):
        return lambda *a, **kw: self.func(*a, **kw, **{self.argname: argvalue})


def template(argname):
    return lambda func: TemplateFunction(func, argname)


@template("x")
def add(y, *, x):
    return x + y


print(add[5](3))
print(add(3, x=5))
```

this works quite well, except when the template function is also a member function:
```py
class C:
    @template("x")
    def add(self, y, *, x):
        return x + y


print(C().add[5](3))  # C.add() missing 1 required positional argument: 'y'
```

it took me many prompts for chatgpt to finally understand what i wanted (or did i give up and found the solution on stackoverflow? i forgor) -- apparently theres something called a descriptor, something something, `__get__` solves all of our problems:
```py
class TemplateFunction:
    ...

    def __get__(self, instance, _):
        return TemplateFunction(
            lambda *a, **kw: self.func(instance, *a, **kw), self.argname
        )
```

this way when `instance.template_func` is called, `instance` is passed to the `__get__` first where we can capture it before it is lost forever

written so far is my recollection of how i arrived at [`ParametrizedFunc`](https://github.com/hljeong/py_utils/blob/main/py_utils/parametrize.py), which is a more polished version of the code above. while writing this post ive also discovered a despicable way to reference undefined names:
```py
import types


class TemplateFunction:
    def __init__(self, func, argname):
        self.func = func
        self.argname = argname

    def __getitem__(self, argvalue):
        return types.FunctionType(self.func.__code__, {self.argname: argvalue})


def template(argname):
    return lambda func: TemplateFunction(func, argname)


@template("x")
def add(y):
    return x + y


print(add[5](3))
```

somehow i feel sick...
