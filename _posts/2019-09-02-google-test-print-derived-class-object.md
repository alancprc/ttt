---
layout:     post
title:      google test 打印派生类对象
categories: c++
---

在Unison中使用google test时，发现EXPECT_EQ在fail时，不能打印Unison Test Language中定义的派生类的对象。于是写了个纯C++的示例，发现在只定义基类的`operator<<`时，无法打印派生类对象。

于是在github上给googletest提交了一个issue[Unable to print derived class while `operator<<` for base class defined #2435](https://github.com/google/googletest/issues/2435)。
内容如下：

I'm new to use google test and not sure it's a problem with google test or the
way I use it.

`class D` is derived from `class B`, and `operator<<` for `class B` is defined.
with this, `std::cout` is able to print B and D, as shown in test `PrintByCout`.
However, when two object of `class D` is compared using `EXPECT_EQ`, it just
print the byte data instead of using `operator<<` for `class B`.

Only when `operator<<` for `class D` is defined `class D` objects are properly
printed.

The code for testing is shown as follows, along with the execution result.
Tested on v1.8.0/v1.8.1/1.8.x/master, with the same result.

```
#include <iostream>
#include <string>

class B {
 public:
  B(std::string name = "class B") : name_(name) {}
  bool operator==(const B& rhs) const { return name_ == rhs.Name(); }
  std::string Name()const { return name_;}

 private:
  std::string name_;
};

class D : public B {
 public:
  D(std::string name = "class D") : B(name) {}
};

std::ostream& operator<<(std::ostream& os, const B& b) {
  os << b.Name();
  return os;
}

/* without this operator<< for D, EXPECT_EQ won't print D properly */
/*
std::ostream& operator<<(std::ostream& os, const D& b) {
  os << b.Name();
  return os;
}
*/

#include <gtest/gtest.h>

TEST(Print, PrintByCout) { std::cout << B() << std::endl << D() << std::endl; }

TEST(Print, PrintB) {
  B obj1("b1");
  B obj2("b2");
  EXPECT_EQ(obj1, obj2);
}

TEST(Print, PrintD) {
  D obj1("d1");
  D obj2("d2");
  EXPECT_EQ(obj1, obj2);
}
```
output :
```
Running main() from ./googletest/googletest/src/gtest_main.cc
[==========] Running 3 tests from 1 test case.
[----------] Global test environment set-up.
[----------] 3 tests from Print
[ RUN      ] Print.PrintByCout
class B
class D
[       OK ] Print.PrintByCout (0 ms)
[ RUN      ] Print.PrintB
sample.cpp:39: Failure
Expected equality of these values:
  obj1
    Which is: b1
  obj2
    Which is: b2
[  FAILED  ] Print.PrintB (0 ms)
[ RUN      ] Print.PrintD
sample.cpp:45: Failure
Expected equality of these values:
  obj1
    Which is: 32-byte object <90-64 84-4C FF-7F 00-00 02-00 00-00 00-00 00-00 64-31 00-6E 39-56 00-00 E0-E1 75-6E 39-56 00-00>
  obj2
    Which is: 32-byte object <B0-64 84-4C FF-7F 00-00 02-00 00-00 00-00 00-00 64-32 00-4C FF-7F 00-00 40-7D D9-7F 87-7F 00-00>
[  FAILED  ] Print.PrintD (0 ms)
[----------] 3 tests from Print (0 ms total)

[----------] Global test environment tear-down
[==========] 3 tests from 1 test case ran. (0 ms total)
[  PASSED  ] 1 test.
[  FAILED  ] 2 tests, listed below:
[  FAILED  ] Print.PrintB
[  FAILED  ] Print.PrintD
```

没想到很快就有人回复了，[kuzkry](https://github.com/kuzkry)通过一小段代码说明了原因：
```
#include <iostream>

struct B {};
struct D : B {};

template <typename T>
void foo(T) {
    // This is something that works under the hood of GTest.
    // I think we cannot change this as we don't know what types
    // users will try to print.
    std::cout << "Not what we wanted\n";
}

void foo(const B&) {
    // This is what we want.
    std::cout << "Good\n";
}

int main() {
    foo(D{}); // oops, a function template is a better match
}
```
并且也还提供了解决办法：
```
/* template header */
template <typename T, typename = typename std::enable_if<std::is_base_of<B, T>::value>::type>  // (C++11)
template <typename T, typename = std::enable_if_t<std::is_base_of<B, T>::value>>  // (C++14)
template <typename T, typename = std::enable_if_t<std::is_base_of_v<B, T>>>  // (C++17)
std::ostream& operator<<(std::ostream& os, const T& b) {
  os << b.Name();
  return os;
}
```
完美！

