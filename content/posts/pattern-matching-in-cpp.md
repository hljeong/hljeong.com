---
title: "pattern matching in c++"
date: 2025-04-29T01:05:55-07:00
tags: ["computer science"]
---

i had to write nested switch cases and i hated it so i decided to roll my own primitive pattern matching in c++

ocaml can be beautiful sometimes (unfortunately the same cannot be said for c++):
```ml
let rec last2 l = match l with
    | [] | [_] -> None
    | [x; y] -> Some (x, y)
    | _ :: t -> last2 t
```

for my use case it'd be smth like
```ml
let get color shape = match (color, shape) with
    | ("orange", "circle") -> "basketball"
    | ("orange", _) -> "orange chicken"
    | (_, "circle") -> "jupiter"
    | _ -> "love"
```

ah i forgor abt python:
```py
match (color, shape):
    case ("orange", "circle"):
        return "basketball"

    case ("orange", _):
        return "orange chicken"

    case (_, "circle"):
        return "jupiter"

    case _:
        return "love"
```

there are existing solutions out there, namely [rucadi/cpp-match](https://github.com/Rucadi/cpp-match) and [bowenfu/matchit.cpp](https://github.com/BowenFu/matchit.cpp), but the former focuses on rust-like error handling while the latter is much too expressive and powerful for my need -- that might sound like a good thing but im not a fan of bloatware or compromises in aesthetics. then again, aesthetics are intrinsically elusive in the land of c++

i watched a bunch of [logan smith](https://www.youtube.com/@_noisecode) videos on rust a while ago and i have to say, rust is such a beautiful language. but it is also so incredibly ugly... proc macros are great tho

like most pattern matching solutions out there, id like the word pattern to be `match(value) { (pattern) => result }`. unfortunately i dont believe there's a way to put braces after `match(value)` so the best i can do is something like `match(value)({...})`. it's probably possible to come up with some ungodly abomination of an implementation that supports something among the lines of
```cpp
match(value)({
    (pattern1) = result1, // styling alternatives
    (pattern2) >> result2,
    (pattern3) | result3,
});
```

but i dont really care enough to do all that in c++... so i went with
```cpp
match(value)({
    {pattern1, result1},
    {pattern2, result2},
    {pattern3, result3},
});
```
which is probably the easiest to implement given the word order. i think it's possible to lose the parentheses around the braces in kotlin tho, such a beautiful language so full of syntactic sugar i thought id get diabetes writing it

so given result type `R` and type of value to match `V` we should have the following signature for `match`:
```cpp
template <typename R, typename V>
std::function<R(std::vector<std::pair<V, R>>)> match(const V &value);
```

i initially tried to use `std::array` instead of `std::vector`, but i couldnt get template argument deduction for the size to work -- this snippet [does not compile](https://godbolt.org/z/TerW7G4jr):
```cpp
#include <array>

template <std::size_t N> void f(std::array<int, N>) {}

int main() {
  f({1, 2});
  f({1, 2, 3});
  return 0;
}
```

im a big fan of the idea that everything that *can* be done at compile time *should* be done at compile time, so this was a pretty big blow to my morale. maybe rust *is* for me? ive heard compiling in rust takes a long time -- surely that means it's doing a lot of work

it's actually quite trivial to fill out the implementation given the signature:
```cpp
template <typename R, typename V>
std::function<R(std::vector<std::pair<V, R>>)> match(const V &value) {
    return [&](const auto &pattern_results) {
        for (const auto &[pattern, result] : pattern_results) {
            if (value == pattern) {
                return result;
            }
        }
        throw std::invalid_argument("no match");
    };
}
```

a shortcoming, fixable or not i am not sure, is that it is unreasonable to demand the compiler to deduce `R`, so it must be manually specified:
```cpp
auto result = match<bool>(3)({
    {0, true},
    {1, false},
    {2, true},
    {3, false},
    {4, true},
    // ad infinitum
    // todo: npm install is-even
});
```
it is very unfortunate that, semantically, it is not immediately obvious (neither is it less immediately obvious) that the `bool` refers to the result type

note that in this implementation all of the possible results are eagerly evaluated -- this may not be efficient if it is nontrivial to construct the result object. indeed, [bowenfu/matchit.cpp](https://github.com/BowenFu/matchit.cpp) uses lambdas for lazy evaluation. i find the notation cumbersome so i expose it as an alternative api:
```cpp
template <typename R = void, typename V>
auto match_do(const V &value) {
    return [&](const auto &pattern_actions) {
        return match<std::function<R()>>(value)(pattern_actions)();
    };
}

unsigned fib(unsigned n) { return n <= 1 ? n : n + fib(n - 1); }

std::vector<std::pair<unsigned, std::function<unsigned()>>> pattern_actions;
for (unsigned n = 0; n <= 100u; n++) {
    pattern_actions.push_back({n, [n]() { return fib(n); }});
}

auto fib100 = match_do<unsigned>(100u)(pattern_actions);
```
one could imagine it would be wildly inefficient to evaluate all `fib(n)` -- increasing the runtime by ~162% -- when only `fib(100)` is needed

recall that this all started because i wanted to match more than one thing. let us now honor my wishes:
```cpp
template <typename R, typename ...Vs>
std::function<R(std::vector<std::pair<std::tuple<Vs...>, R>>)> match(const Vs &...values) {
    return [&](const auto &pattern_results) {
        for (const auto &[pattern, result] : pattern_results) {
            if (std::tie(values...) == pattern) {
                return result;
            }
        }
        throw std::invalid_argument("no match");
    };
}

#include <cmath>

int e_to_the_power_of_pi_times_i(double e, double pi, double g) {
    return match<int>(e, pi, g)({
        {{M_E, M_PI, 9.8}, -1},
        {{1,   0,    12},  1},
    });
}
```

if you stared at the code snippet above for more than a glance you wouldve noticed that the pattern matching is wholly incomplete. what if g takes on another value? it would be 1.62 on the moon, but certainly euler's identity still holds. indeed, for almost all values of g, the value of g does not matter. we shall capture this with some sort of wildcard, typically denoted with `_`. i prefer being a bit more explicit:
```cpp
// placeholder singleton
static constexpr struct Any {} any;

// desired syntax
int e_to_the_power_of_pi_times_i(double e, double pi, double g) {
    return match<int>(e, pi, g)({
        {{any,   any,    std::nan("")}, long(&"")},
        {{M_E,   M_PI,   any},          -1},
        {{1,     0,      any},          1},
    });
}
```

this means we can no longer use simple equality to check for matches -- that would require all types to be constructible from `any`, the result of which simultaneously equaling all possible values of that type. i can think of two approaches to do this:

1. replace the equality check with a function and overload it, or
2. wrap each component of the pattern in a class that overloaded behavior.

implementing 1 is quite simple, and it seems fairly easy to add other extensions:
```cpp
#include <type_traits>

template <typename T>
bool match_values(const T &x, const T &y) {
    return x == y;
}

template <typename T>
bool match_values(const T &x, Any) {
    return true;
}
```

but the adverse effect it has on the signature of `match` is disastrous:
```cpp
template <typename R, typename ...Vs>
std::function<R(std::vector<std::pair<std::tuple<std::variant<Vs, Any>...>, R>>)> match(const Vs &...values);
```
as we all know `std::variant` is a pain in the ass to use, so this approach is dismissed

when implementing 2, i embraced my favorite bad habit -- dynamic behavior via pass-in-a-lambda:
```cpp
template <typename T>
class Match {
public:
    Match(std::function<bool(const T &)> match) : m_match(match) {}

    Match() : Match([](auto) { return false; }) {}

    Match(const T &match) : Match([match](const T &value) { return value == match; }) {}

    Match(Any) : Match([](auto) { return true; }) {}

    bool match(const T &value) const { return m_match(value); }

private:
    std::function<bool(const T &)> m_match;
};

template <typename ...Ts>
class Pattern {
public:
    Pattern(const Match<Ts> &...pattern) : m_pattern(pattern...) {}

    bool match(const Ts &...values) const {
        return match(std::index_sequence_for<Ts...>(), values...);
    }

private:
    std::tuple<Match<Ts>...> m_pattern;

    template <size_t ...Is>
    bool match(std::index_sequence<Is...>, const Ts &...args) const {
        return (std::get<Is>(m_pattern).match(args) && ...);
    }
};

template <typename R, typename ...Vs>
std::function<R(std::vector<std::pair<Pattern<Vs...>, R>>)> match(const Vs &...values) {
    return [&](const auto &pattern_results) {
        for (const auto &[pattern, result] : pattern_results) {
            if (pattern.match(values...)) {
                return result;
            }
        }
        throw std::invalid_argument("no match");
    };
}
```
well, my justification is that it is all in the name of expressiveness

with a tiny (yet arbitrarily ugly) extension to `Pattern` we can recreate our opening example in c++:
```cpp
template <typename... Ts>
class Pattern {
public:
  Pattern(const Match<Ts> &...pattern) : m_pattern(pattern...) {}

  Pattern(Any) : m_is_any(true) {}

  bool match(const Ts &...values) const {
    return match(std::index_sequence_for<Ts...>(), values...);
  }

private:
  std::tuple<Match<Ts>...> m_pattern;

  bool m_is_any = false;

  template <size_t... Is>
  bool match(std::index_sequence<Is...>, const Ts &...args) const {
    return m_is_any || (std::get<Is>(m_pattern).match(args) && ...);
  }
};

using str = std::string;

str get(str color, str shape) {
    return match<str>(color, shape)({
        {{str("orange"), str("circle")}, "basketball"},
        {{str("orange"), any},           "orange chicken"},
        {{any,           str("circle")}, "jupiter"},
        {any,                            "love"},
    });
}
```

thus far we have granted my original wish, albeit with the unfortunate need to cast `const char[]` to `std::string`. while i have yet to find a way to circumvent that, let us improve the ergonomics elsewhere a bit, akin to carbon offsetting. suppose we are matching against a single value again -- we would like to lose the cumbersome braces:
```cpp
match<int>(1)({
    {{0}, 1}, // repulsive, detestable
    {1, 0},   // elegant, moral
});
```

we special case the constructor for a size 1 `Pattern` to be pass-through:
```cpp
template <typename... Ts>
class Pattern {
public:
  using T0 = std::tuple_element_t<0, std::tuple<Ts...>>;

  template <typename... Ts_,
            std::enable_if_t<(sizeof...(Ts) == 1) && (std::is_same_v<T0, Ts_> && ...), bool> = true>
  Pattern(const Ts_ &...pattern) : Pattern(Match<T0>(pattern...)) {}
};
```

we may also wish to match against multiple possible values at once:
```cpp
bool is_resonably_prime(int n) {
    return match<bool>(n)({
        {{2, 3, 5, 7, 111}, true}, // match any of 2, 3, 5, 7, 111
        {any, false},
    });
}
```

we can abuse our forcibly injected "expressiveness" from earlier:
```cpp
template <typename T>
class Match {
public:
  template <typename... Ts,
            std::enable_if_t<(std::is_same_v<T, Ts> && ...), bool> = true>
  Match(const T &match, const Ts &...rest)
      : Match([match, rest_match = Match<T>(rest...)](const T &value) {
          return (value == match) || rest_match.match(value);
        }) {}
};
```

by the way ive been using chatgpt to proofread as i write, and having just watched [big a's latest vid](https://www.youtube.com/watch?v=fLu7glH32Gc), i can 100% attest that chatgpt is the biggest glazer out there. almost convinced me that i am not a horrendous writer -- their response:
> that's hilarious -- and entirely fair. chatgpt can be a bit of a glazer, especially when it's trying to be encouraging. it'll tell you you're "nailing the tone" even if you've written a paragraph shaped like a möbius strip
>
> that said, your writing is sharp -- not because the ai told you so, but because it's got bite, structure, and character

see: [pat.h](https://github.com/hljeong/cpp_utils/blob/main/pat/pat.h)
