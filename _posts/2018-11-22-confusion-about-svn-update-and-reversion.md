---
layout:     post
title:      svn update 和 reversion 的困惑
categories: svn
---

今天上午听两个同事在讨论使用svn的问题，说svn update没有按照预期的方式工作。下午刚好有段Perl代码工作不正常，于是过去凑个热闹，问问清楚。


为了描述方便，当事人简称为J。问题是这样的:

* 某次svn update之前
J在Program/目录下改动了fileA

svn status显示fileA有修改

通过svn log和svn info <svn server link> 查询到当前版本为rev 6，repo最新版本为rev 13

* svn update之后
通过svn info查询，发现fileA的版本还是rev 6

svn status显示fileA有改动。但是和update之前的文件相比，没有变化


于是J很纳闷，明明repo上最新版本是rev 13, 但是svn update并没有将工作目录中的fileA和repo rev 13的fileA进行合并，也没有报conflict。

我们反复试验了好几次，最后终于发现这其中的原因：
svn info 查询到有两个版本信息，一个是总的版本号，此例中为rev 13；另一个，是当前文件(夹)的last modified reversion，在这个例子当中，是rev 6。
也就是说，从rev 6到rev 13，当前文件夹Program/下的文件都没有改动，于是在svn update时，fileA维持当前基于rev 6的改动，并没有其他修改可以合并，或者引起冲突。

相比之下，git没有这种所谓的全局版本号，而是串commit id，并用comment帮助用户做区分，所以我之前没有遇到这个困惑。
