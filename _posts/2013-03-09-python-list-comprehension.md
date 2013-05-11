---
layout: post
title: Python 的 list/generator comprehension 哲学
category: tech
tags: [python]
---

早就想对最近的一些应用在python上的functional programming 发表一些看法了。Functional programming确实很好很实用，但是reduce，filter，map这些函数在python里是可有可无的，很多人为了显示自己的functional programming功底，有意或无意的过多的使用这些函数，而忽略了python强大的 list/generator comprehension。

python的作者Guido在最初设计py3k的时候设想把filter和map这些function拿掉。在拥有list/generator comprehension的python里，这些函数存在的最大意义大概是为了照顾函数式编程忠实粉丝的使用习惯吧。

举个例子，看看在python里实现filter和map是一件多么简单的事情吧。

假设我么要从nums这个数组里找出所有大于0的数：

    [x for x in nums if x >0]
对nums里所有的数值进行F操作：

    [F(x) for x in nums]
这只是最简单的例子，是不是也能看出比filter和map等函数更intuitive？

好吧也许你还不能看出reduce怎么被取代，reduce的功能不过是在map的基础上增加了个sum操作而已：


    sum([x+1 for x in nums])
sum只是一个示范，你可以替换成任意实际中需要的函数。

函数式编程的确很强大，但它的强大不仅仅体现在用几个filter，map之类的函数而已。