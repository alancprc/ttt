---
layout:     post
title:      Perl error: install_driver(mysql) failed
categories: Perl
---

最近到客户那写点Perl代码，并用`pp`将Perl代码和依赖的modules打包成一个可执行文件。在执行时遇到一个错误：
```
install_driver(mysql) failed: Can't load '/tmp/par...*so' for module DBD::mysql: libmysqlclient.so.16: cannot open shared object file: No such file or directory at /usr/lib64/perl5/DynaLoader.pm line 200.
```
觉得很奇怪，系统上的`mysqlclient.so`的版本是`libmysqlclient.so.20`并不是`libmysqlclient.so.16`，不知哪个地方明确非要这个版本，一时没了头绪。但仔细看报错信息，显然和`DBD::mysql`相关，于是找到`DBD::mysql`对应的`mysql.so`，用`ldd`查看`mysql.so`的信息，发现正是它缺少了这几个依赖。
由此推测，`pp`打包的一大堆modules是从其它机器上直接复制过来，而不是`cpan/cpanm`重新安装的。而且这两个系统的`mysql`版本不一样，导致`libmysqlclient.so`的版本也不一致，源机器上的`.so`版本在本机上找不到，于是报错。
于是重新安装`DBD::mysql`，新的`mysql.so`依赖的就是系统现有的`libmysqlclient.so.20`了。

结论就是，`cpan/cpanm`安装的modules，只有在系统相同的情况下，可以直接copy到其他机器上，否则要在目标系统上重新安装。
不过好在，`cpanm`有个`--save-dists`的选项，可以保存module的安装包。
