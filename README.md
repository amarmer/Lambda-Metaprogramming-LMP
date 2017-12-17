# Recursive Lambda and Metaprogramming

How in a function to create std::array on stack with `contexpr int N`.
It is is obviuos how to do in a template function, for instance like:

```C++
template <int SIZE>
void Test() { std::array<SIZE> arr; }

Test<10>();
```
How to do it in lambda function?

```C++
auto test = [](auto size) { std::array<size> arr; }

test(std::integral_contant<int, 10>());
```
`integral_constant` looks like:
```C++
template<class T, T val>
struct integral_constant {	
	static constexpr T value = val;
  
  constexpr operator value_type() const {	
		return (value);
	}
};
```


Let's create a struct `ConstInt`:
```C++
template <int N>
struct ConstInt: public integral_constant<int, N> {
  template <int INC>
  constexpr auto plus() { 
    return ConstInt<N + INC>(); 
  }
};
```

