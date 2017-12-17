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

Let's create a struct `ConstInt` and template function `RecusrsiveLambda':
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
```

Bellow is how a tuple can be enumerated and it's elements are printed out:
```C++
RecursiveLambda(
  [&tpl](auto lambda, auto index) {
    if constexpr(index < std::tuple_size<typename std::decay<TPL>::type>::value) {
      std::cout << std::get<index>(tpl) << std::endl;

      lambda(lambda, index.plus<1>());
    }
  }
);
```

Example how a new tuple with reversed elements can be created:

```C++
auto var = RecursiveLambda(
  [&tpl](auto f, auto index, auto&&...args) {
    if constexpr(0 == sizeof...(args))
      return f(f, index.plus<1>(), std::make_tuple(std::get<index>(tpl)));
    else {
      if constexpr(index < TupleSize<decltype(tpl)>())
        return f(f, index.plus<1>(), std::tuple_cat(std::make_tuple(std::get<index>(tpl)), args...));
      else
        return std::forward<decltype(args)...>(args...);
    }
  }
 ); 
```

And the same code can be simplified if added `tuple<>()` after lambda function:
```C++
auto var = RecursiveLambda(
  [&tpl](auto lambda, auto index, const auto& curTpl) {
    if constexpr(index < TupleSize<decltype(tpl)>())
      return lambda(lambda, index.plus<1>(), std::tuple_cat(std::make_tuple(std::get<index>(tpl)), curTpl));
    else
      return curTpl;
  },
  std::tuple<>()
);
```
