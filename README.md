# Lambda Metaprogramming (LMP)

#### Word `metaprogramming` is used together with word `template` - `template metaprogramming (TMP)`. 
#### Here is described that `metaprogramming` can be used with `lambda` as well - `lambda metaprogramming (LMP)`. 

This is a classic example of calculating factorial using TMP:
```C++
template <unsigned int n>
constexpr int factorial() {
  if constexpr(0 == n)
    return 1;
  else
    return n * factorial<n - 1>();
};
```

Bellow is implementation of factorial using LMP.

Since lambda doesn't have name, in order to simplify calling it recursively, bellow is a helper function `RecursiveLambda`.

```C++
template <typename FUNC, typename ...ARGS>
constexpr auto RecursiveLambda(FUNC lambda, ARGS&&... args) { return lambda(lambda, args...); }
```

```C++
auto factorial = [](auto n) {
  return RecursiveLambda(
    [](auto lambda, auto n) {
      if constexpr(0 == n)
        return 1;
      else
        return n * lambda(lambda, std::integral_constant<int, n - 1>());
    },
    n
  );
};

constexpr auto factorial_5 = factorial(std::integral_constant<int, 5>());
```

Notice that `(0 == n)` compiles because `std::integral_constant` has `constexpr` cast operator to `int`.

Let's look at more LMP examples with `tuple`.

Bellow is a helper function `TupleSize` wich simplifies getting size of a tuple.

```C+ +
template <typename TPL>
constexpr auto TupleSize() { return std::tuple_size<typename std::decay<TPL>::type>::value; }
```

And a helper alias `IntegralConstant`.

```C+ +
template <auto N>
using IntegralConstant = std::integral_constant<decltype(N), N>;
```

Example of how a tuple can be enumerated and it's elements are printed out:

```C+ +
RecursiveLambda(
  [&tpl](auto lambda, auto index) {
    if constexpr(index < TupleSize<decltype(tpl)>()) {
      std::cout << std::get<index>(tpl) << std::endl;

      lambda(lambda, IntegralConstant<index + 1>());
    }
  },
  IntegralConstant<0>());
);
```

Example of how a new tuple with reversed elements can be created :

```C++
auto reversedTpl = RecursiveLambda(
  [&tpl](auto lambda, auto index, auto&&...args) {
    if constexpr(0 == sizeof...(args))
      return lambda(lambda, IntegralConstant<index + 1>(), std::make_tuple(std::get<index>(tpl)));
    else {
      if constexpr(index < TupleSize<decltype(tpl)>())
        return lambda(lambda, IntegralConstant<index + 1>(), std::tuple_cat(std::make_tuple(std::get<index>(tpl)), args...));
      else
        return std::forward<decltype(args)...>(args...);
    }
  },
  IntegralConstant<0>()
);
```

Code above can be simplified if added `tuple<>()` parameter after lambda function :
```C++
auto reversedTpl = RecursiveLambda(
  [&tpl](auto lambda, auto index, const auto& curTpl) {
    if constexpr(index < TupleSize<decltype(tpl)>())
      return lambda(lambda, IntegralConstant<index + 1>(), std::tuple_cat(std::make_tuple(std::get<index>(tpl)), curTpl));
    else
      return curTpl;
  },
  IntegralConstant<0>(),
  std::tuple<>()
);
```

Example of how to cancatenate 2 tuples:
```C++
auto catTpl = RecursiveLambda(
  [&tpl1, &tpl2](auto lambda, auto index, auto&&...args) {
    constexpr auto total = TupleSize<decltype(tpl1)>() + TupleSize<decltype(tpl2)>();
    if constexpr(0 == total)
      return std::tuple<>();
    else {
      auto getEl = [&](auto index) {
        if constexpr(index < TupleSize<decltype(tpl1)>())
          return std::get<index>(tpl1);
        else
          return std::get<index - TupleSize<decltype(tpl1)>()>(tpl2);
      };

      if constexpr(0 == sizeof...(args))
        return lambda(lambda, IntegralConstant<index + 1>(), getEl(index));
      else
        if constexpr(index < total)
          return lambda(lambda, IntegralConstant<index + 1>(), args..., getEl(index));
        else
          return std::make_tuple(args...);
    }
  },
  IntegralConstant<0>()
);
```
