# Recursive Lambda and Metaprogramming

How in a function to create std::array on stack with `contexpr int N`.
It is is obviuos how to do in a template function, for instance like:

```C++
template <int SIZE>
void Test() { std::array<SIZE> arr; }

Test<10>();
```
And this how to do it in lambda function:

```C++
auto test = [](auto size) { std::array<size> arr; }

test(std::integral_contant<int, 10>());
```
It works because there a cast operator to `int` in `integral_constant` which looks like:
```C++
template<class T, T val>
struct integral_constant {	
  static constexpr T value = val;
  
  constexpr operator T() const { return (value); }
};
```

Then in a lambda function an argument can be used similarly to explicit template parameter in a template function.

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

