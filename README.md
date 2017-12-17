# Recursive Lambda and Metaprogramming

Bellow is example how in a lambda function to create std::array on stack with `contexpr int N`:

```C++
auto test = [](auto size) { std::array<decltype(size)::value_type, static_cast<size_t>(size)> arr; };

test(std::integral_contant<int, 10>());
```
It works because there a cast operator to `int` in `integral_constant` which looks like:
```C++
template<class T, T val>
struct integral_constant {	
  static constexpr T value = val;
	
  using value_type = T;
  
  constexpr operator T() const { return (value); }
};
```


It is a generic approach when in a lambda function an argument can be used similarly to explicit template parameter in a template function.

Let's create a struct `ConstInt` and template function `RecursiveLambda':
```C++
template <int N>
struct ConstInt: public integral_constant<int, N> {
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

Bellow is how a tuple can be enumerated and it's elements are printed out:
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

Example how a new tuple with reversed elements can be created:

```C++
auto newTpl = RecursiveLambda(
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

And the same code can be simplified if added `tuple<>()` parameter after lambda function:
```C++
auto newTpl = RecursiveLambda(
  [&tpl](auto lambda, auto index, const auto& curTpl) {
    if constexpr(index < std::tuple_size<typename std::decay<TPL>::type>::value)
      return lambda(lambda, index.plus<1>(), std::tuple_cat(std::make_tuple(std::get<index>(tpl)), curTpl));
    else
      return curTpl;
  },
  std::tuple<>()
);
```

Example how to cancatenate 2 tuples:
```C++
auto newTpl = RecursiveLambda(
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
