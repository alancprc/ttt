---
layout: post
title: "markdown test"
categories: test
---

title:

first level header
========

second level header
--------

items:

* item 1
    * item 2

1. point 1
    2. point 2

web link:
I like [Google](https://www.google.com "Google it!") (this is a in-line link.)

web link2:
I like [Google][1] (this is a reference-style link.)

image link:
a nice pic ![windows 10 sketch]({{ site.baseurl }}/pic/Windows_10_Sketch.png)

bold:
**hello world!**

italiy:
_this is my second post on the github page._


code highlight:
{% highlight c++ %}
#include <iostream>
int main()
{
    std::cout << "hello world!" << endl;
    return 0;
}
{% endhighlight %}

code:
linux command:

    systemctl start autofs
    systemctl enable autofs

c++ code:

    #include <iostream>
    int main()
    {
        std::cout << "hello world!" << endl;
        return 0;
    }


[1]: https://www.google.com/ "Optical title here"
