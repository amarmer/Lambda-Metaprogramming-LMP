# Lambda Metaprogramming (LMP)

##### Word `metaprogramming` is used together with word `template` - `template metaprogramming (TMP)`. 
##### Here is described that `metaprogramming` can be used with `lambda` as well - `lambda metaprogramming (LMP)`. 

For instance in a template function `foo` allocates array on stack using `std::array`.
```C++
template <int SIZE>
void foo() {
  std::array<char, SIZE> arr;
}

foo<100>();
```

This is how it can be implemented using lambda.
```C++
auto foo = [](auto size) {
  std::array<char, size> arr;
};
    
foo(std::integral_constant<int, 100>());
```
`std::array<char, size>` compiles because `size` is `std::integral_constant` which has `constexpr` cast operator to `int`.

TMP is often used with recursion. For instance, a classic example of calculating factorial using TMP.
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

Lambda doesn't have a name, and it is not possible to call it recursively.<br/>
In order to call lambda recursively, a helper function `RecursiveLambda` is used.

```C++
template <typename LAMBDA>
constexpr auto RecursiveLambda(LAMBDA lambda)
{
  return [lambda](auto&&... args)
  {
    return lambda(lambda, std::forward<decltype(args)>(args)...);
  };
}
```

Factorial implementation using `RecursiveLambda`.
```C++
auto factorial = RecursiveLambda(
  [](auto lambda, auto n) {
    if constexpr(n == 0)
      return 1;
    else
      return n * lambda(lambda, IntegralConstant<n - 1>());
  }
);

constexpr auto factorial_5 = factorial(std::integral_constant<int, 5>());
```

Bellow are LMP examples with `tuple`.

`TupleSize` is a helper function which simplifies getting size of a tuple.
```C++
template <typename TPL>
constexpr auto TupleSize() { return std::tuple_size<typename std::decay<TPL>::type>::value; }
```

And `IntegralConstant` is a helper alias for `std::integral_constant`.
```C++
template <auto N>
using IntegralConstant = std::integral_constant<decltype(N), N>;
```

Example of lambda function `printLambda` which enumerates a `tuple` and converts each tuple's element to `string`.<br/> 
Then this function can be used to print tuple's elements using `format` lambda which is passed as a parameter.

```C++
std::function<void(std::function<void(int, const std::string&)>)> printLambda;

printLambda = [&tpl](auto format) {
  RecursiveLambda(
    [](auto lambda, const auto& tpl, auto format, auto index) {
      if constexpr(index < TupleSize<decltype(tpl)>()) {
        std::stringstream strStream;
        strStream << std::get<index>(tpl);

        format(index, strStream.str());

        lambda(lambda, tpl, format, IntegralConstant<index + 1>());
      }
    }
  )(tpl, format, IntegralConstant<0>());
};

printLambda([](int index, const std::string& tupleEl) {
  std::cout << "tuple[" << index << "]: " << tupleEl << std::endl;
});
```

Example of how a new tuple with reversed elements can be created.
```C++
auto reversedTpl = RecursiveLambda(
  [&tpl](auto lambda, auto index, auto&&...args) {
    if constexpr(0 == sizeof...(args))
      return lambda(lambda, IntegralConstant<index + 1>(), std::make_tuple(std::get<index>(tpl)));
    else {
      if constexpr(index < TupleSize<decltype(tpl)>())
        return lambda(lambda, 
                      IntegralConstant<index + 1>(), 
                      std::tuple_cat(std::make_tuple(std::get<index>(tpl)),
                                     std::forward<decltype(args)>(args)...));
      else
        return std::forward<decltype(args)...>(args...);
    }
  }
)(IntegralConstant<0>());
```

Code above can be simplified if added second parameter `std::tuple<>()`.
```C++
auto reversedTpl = RecursiveLambda(
  [&tpl](auto lambda, auto index, const auto& curTpl) {
    if constexpr(index < TupleSize<decltype(tpl)>())
      return lambda(lambda, 
                    IntegralConstant<index + 1>(), 
                    std::tuple_cat(std::make_tuple(std::get<index>(tpl)), curTpl));
    else
      return curTpl;
  }
)(IntegralConstant<0>(), std::tuple<>());
```

Example of how to cancatenate 2 tuples.
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
          return lambda(lambda, 
                        IntegralConstant<index + 1>(), 
                        std::forward<decltype(args)>(args)..., 
                        getEl(index));
        else
          return std::make_tuple(std::forward<decltype(args)>(args)...);
    }
  }
)(IntegralConstant<0>());
```

After posting this arcticle on reddit/cpp, reddit.com/user/dima_mendeleev suggested a simplification,<br/>
with which no need to call `lambda(lambda, ...)`, just `lambda(...)`.
It compiles only with GCC 7.2.0.
```C++
template <typename LAMBDA>
constexpr auto RecursiveLambda(LAMBDA lambda)
{
  return [lambda](auto&&... args)
  {
    return lambda(RecursiveLambda(lambda), std::forward<decltype(args)>(args)...);
  };
}
```

With modified `RecursiveLambda` function above, which creates reversed `tuple`, it looks like:
```C++
auto reversedTpl = RecursiveLambda(
  [&tpl](auto lambda, auto index, const auto& curTpl) {
    if constexpr(index < TupleSize<decltype(tpl)>())
      return lambda(IntegralConstant<index + 1>(),
                    std::tuple_cat(std::make_tuple(std::get<index>(tpl)), curTpl));
    else
      return curTpl;
  }
)(IntegralConstant<0>(), std::tuple<>());
```

It would be usefull in C++ to have keyword `lambda` which allows to refer to a lambda function from inside the function,<br/>
similar to the keyword `this` inside a class. Then code above could look like:
```C++
auto reversedTpl = [&tpl](auto index, const auto& curTpl) {
  if constexpr(index < TupleSize<decltype(tpl)>())
    return lambda(IntegralConstant<index + 1>(), 
                  std::tuple_cat(std::make_tuple(std::get<index>(tpl)), curTpl));
  else
    return curTpl;
}(IntegralConstant<0>(), std::tuple<>());
```
