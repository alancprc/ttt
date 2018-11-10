---
layout:     post
title:      undefined symbol, DSO missing error, when compile with googletest
categories: googletest,c++
---

最近在学习google test, 准备在工作中写的一些C++代码引入单元测试。

一开始按照google test的文档进行对sample进行编译测试没有问题，但是将google test放在用户目录，似乎编译的时候稍微麻烦了点。刚好arch中有gtest的package，就安装到了系统目录/usr和/lib。
但是在利用系统路径中的gtest编译sample的时遇到了错误：
```
g++ -g -Wall -Wextra -pthread -lpthread sample1.o sample1_unittest.o /lib/libgtest_main.so -o sample1_unittest
/usr/bin/ld: sample1_unittest.o: undefined reference to symbol '_ZN7testing8internal6IsTrueEb'
/usr/bin/ld: /usr/lib/libgtest.so.1.8.1: error adding symbols: DSO missing from command line
```
原始的Makefile:
```
GTEST_DIR = ..

USER_DIR = ../samples

CPPFLAGS += -isystem $(GTEST_DIR)/include

CXXFLAGS += -g -Wall -Wextra -pthread

TESTS = sample1_unittest

GTEST_HEADERS = $(GTEST_DIR)/include/gtest/*.h \
                $(GTEST_DIR)/include/gtest/internal/*.h


all : $(TESTS)

clean :
	rm -f $(TESTS) gtest.a gtest_main.a *.o

# Builds gtest.a and gtest_main.a.

# Usually you shouldn't tweak such internal variables, indicated by a
# trailing _.
GTEST_SRCS_ = $(GTEST_DIR)/src/*.cc $(GTEST_DIR)/src/*.h $(GTEST_HEADERS)

# For simplicity and to avoid depending on Google Test's
# implementation details, the dependencies specified below are
# conservative and not optimized.  This is fine as Google Test
# compiles fast and for ordinary users its source rarely changes.
gtest-all.o : $(GTEST_SRCS_)
	$(CXX) $(CPPFLAGS) -I$(GTEST_DIR) $(CXXFLAGS) -c \
            $(GTEST_DIR)/src/gtest-all.cc

gtest_main.o : $(GTEST_SRCS_)
	$(CXX) $(CPPFLAGS) -I$(GTEST_DIR) $(CXXFLAGS) -c \
            $(GTEST_DIR)/src/gtest_main.cc

gtest.a : gtest-all.o
	$(AR) $(ARFLAGS) $@ $^

gtest_main.a : gtest-all.o gtest_main.o
	$(AR) $(ARFLAGS) $@ $^

# Builds a sample test.  A test should link with either gtest.a or
# gtest_main.a, depending on whether it defines its own main()
# function.

sample1.o : $(USER_DIR)/sample1.cc $(USER_DIR)/sample1.h $(GTEST_HEADERS)
	$(CXX) $(CPPFLAGS) $(CXXFLAGS) -c $(USER_DIR)/sample1.cc

sample1_unittest.o : $(USER_DIR)/sample1_unittest.cc \
                     $(USER_DIR)/sample1.h $(GTEST_HEADERS)
	$(CXX) $(CPPFLAGS) $(CXXFLAGS) -c $(USER_DIR)/sample1_unittest.cc

sample1_unittest : sample1.o sample1_unittest.o gtest_main.a
	$(CXX) $(CPPFLAGS) $(CXXFLAGS) -lpthread $^ -o $@

```
改动后的Makefile:
```
GTEST_DIR = /usr

USER_DIR = ..

CPPFLAGS += 

CXXFLAGS += -g -Wall -Wextra -pthread

TESTS = sample1_unittest

GTEST_HEADERS = $(GTEST_DIR)/include/gtest/*.h \
                $(GTEST_DIR)/include/gtest/internal/*.h


all : $(TESTS)

clean :
	rm -f $(TESTS) gtest.a gtest_main.a *.o

# Builds gtest.a and gtest_main.a.

## Usually you shouldn't tweak such internal variables, indicated by a
## trailing _.
#GTEST_SRCS_ = $(GTEST_DIR)/src/*.cc $(GTEST_DIR)/src/*.h $(GTEST_HEADERS)
#
## For simplicity and to avoid depending on Google Test's
## implementation details, the dependencies specified below are
## conservative and not optimized.  This is fine as Google Test
## compiles fast and for ordinary users its source rarely changes.
#gtest-all.o : $(GTEST_SRCS_)
#	$(CXX) $(CPPFLAGS) -I$(GTEST_DIR) $(CXXFLAGS) -c \
#            $(GTEST_DIR)/src/gtest-all.cc
#
#gtest_main.o : $(GTEST_SRCS_)
#	$(CXX) $(CPPFLAGS) -I$(GTEST_DIR) $(CXXFLAGS) -c \
#            $(GTEST_DIR)/src/gtest_main.cc
#
#gtest.a : gtest-all.o
#	$(AR) $(ARFLAGS) $@ $^
#
#gtest_main.a : gtest-all.o gtest_main.o
#	$(AR) $(ARFLAGS) $@ $^

# Builds a sample test.  A test should link with either gtest.a or
# gtest_main.a, depending on whether it defines its own main()
# function.

sample1.o : $(USER_DIR)/sample1.cc $(USER_DIR)/sample1.h $(GTEST_HEADERS)
	$(CXX) $(CPPFLAGS) $(CXXFLAGS) -c $(USER_DIR)/sample1.cc

sample1_unittest.o : $(USER_DIR)/sample1_unittest.cc \
                     $(USER_DIR)/sample1.h $(GTEST_HEADERS)
	$(CXX) $(CPPFLAGS) $(CXXFLAGS) -c $(USER_DIR)/sample1_unittest.cc

sample1_unittest : sample1.o sample1_unittest.o -lgtest_main
	$(CXX) $(CPPFLAGS) $(CXXFLAGS) -lpthread $^ -o $@
```
之前看到gtest和gtest_main的区别是，gtest_main额外多了一个main函数入口，就节省了自己编写main函数了。
所以感到很奇怪。这里不应该报`undefined reference to symbol`的。

后来看了libgtest_main.so的信息，显示依赖于libgtest.so:
```
[alan@Arch _posts]$ ldd /lib/libgtest_main.so
	linux-vdso.so.1 (0x00007ffe6b148000)
	libgtest.so.1.8.1 => /usr/lib/libgtest.so.1.8.1 (0x00007ff230347000)
	libstdc++.so.6 => /usr/lib/libstdc++.so.6 (0x00007ff2301b8000)
	libc.so.6 => /usr/lib/libc.so.6 (0x00007ff22fff4000)
	libgcc_s.so.1 => /usr/lib/libgcc_s.so.1 (0x00007ff22ffda000)
	libpthread.so.0 => /usr/lib/libpthread.so.0 (0x00007ff22ffb9000)
	libm.so.6 => /usr/lib/libm.so.6 (0x00007ff22fe34000)
	/usr/lib64/ld-linux-x86-64.so.2 (0x00007ff2303c0000)
```
这才想明白，还是忘了动态库和静态库的区别。
在静态库的情况下libgtest_main.a直接包含了libgtest.a，而在动态库的情况下，就变成了libgtest_main.so依赖于libgtest.so。
在编译sample1_unittest时，增加`-lgtest`即可编译通过。

```
sample1_unittest : sample1.o sample1_unittest.o -lgtest_main -lgtest
```
