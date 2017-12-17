# Recursive Lambda and Metaprogramming

Lambda function doesn't have explicit template parameters, but it is possible to emulate them by passing arguments in lambda function and interpret them similarly as explicit template parameters in a template function.

For instance, to allocate `array` on stack in a template function:
```C++
template <int N>
void Test() { std::array<int, N> arr; };

Test<10>();
```
This is how it is can be done in a lambda function:

```C++
auto Test = [](auto size) { std::array<int, size.value> arr; };

Test(std::integral_contant<int, 10>());
```
It works because `value` is `constexpr` in `std::integral_constant` bellow:
```C++
template<class T, T val>
struct integral_constant {	
  static constexpr T value = val;
  
  constexpr operator T() const { return (value); }
};
```

Let's create a struct `ConstInt` and template function `RecursiveLambda`:
```C++
template <int N>
struct ConstInt: public std::integral_constant<int, N> {
  template <int INC>
  constexpr auto plus() { 
    return ConstInt<N + INC>(); 
  }
};

template <typename FUNC, typename ...ARGS>
constexpr auto RecursiveLambda(FUNC lambda, ARGS&&... args) { 
  return lambda(lambda, ConstInt<0>(), args...); 
}

template <typename TPL>
constexpr auto TupleSize() { 
  return std::tuple_size<typename std::decay<TPL>::type>::value; 
}

```

Example of how a tuple can be enumerated and it's elements are printed out:
```C++
RecursiveLambda(
  [&tpl](auto lambda, auto index) {
    if constexpr(index < TupleSize<decltype(tpl)>()) {
      std::cout << std::get<index>(tpl) << std::endl;

      lambda(lambda, index.plus<1>());
    }
  }
);
```
Notice that `index` in `std::get<index>(tpl)` is `ConstInt` and it compiles beacuse of cast to `int` in `integral_constant`.

Example of how a new tuple with reversed elements can be created:

```C++
auto reversedTpl = RecursiveLambda(
  [&tpl](auto lambda, auto index, auto&&...args) {
    if constexpr(0 == sizeof...(args))
      return lambda(lambda, index.plus<1>(), std::make_tuple(std::get<index>(tpl)));
    else {
      if constexpr(index < TupleSize<decltype(tpl)>())
        return lambda(lambda, index.plus<1>(), std::tuple_cat(std::make_tuple(std::get<index>(tpl)), args...));
      else
        return std::forward<decltype(args)...>(args...);
    }
  }
 ); 
```

Code above can be simplified if added `tuple<>()` parameter after lambda function:
```C++
auto reversedTpl = RecursiveLambda(
  [&tpl](auto lambda, auto index, const auto& curTpl) {
    if constexpr(index < TupleSize<decltype(tpl)>())
      return lambda(lambda, index.plus<1>(), std::tuple_cat(std::make_tuple(std::get<index>(tpl)), curTpl));
    else
      return curTpl;
  },
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
        return lambda(lambda, index.plus<1>(), getEl(index));
      else
        if constexpr(index < total)
          return lambda(lambda, index.plus<1>(), args..., getEl(index));
        else
          return std::make_tuple(args...);
    }
  }
);

```
