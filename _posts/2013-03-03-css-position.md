---
layout: post
title: CSS position 属性
category: tech
tags: [css]
---

CSS的position属性4个值之间的行为区别比较难以理解，作个总结。

以下是[w3schools](http://www.w3schools.com/cssref/pr_class_position.asp)上的描述：

>VALUE   DESCRIPTION 

>static  Default. Elements render in order, as they appear in the document flow

>absolute    The element is positioned relative to its first positioned (not static) ancestor element 

>fixed   The element is positioned relative to the browser window 

>relative    The element is positioned relative to its normal position, so “left:20″ adds 20 pixels to the element’s LEFT position 

为了方便后面的理解，首先介绍一下offset parent这个概念，通常我们将一个节点的直接上级节点成为其parent，那么offset parent和parent的区别就在于，前者不一定是物理上的parent，也有可能是parent的parent，offset parent是节点定位的参照系。position属性4个值的不同，最大的区别就是寻找offset parent的逻辑不同。

static是默认值，通常标签以父标签为定位基准，如果之前有兄弟节点（并且兄弟节点在同一formatting context下），当前标签会为其让出空间。简单的说，如果一个标签的position属性是static，那么该标签的offset全权交给浏览器去计算，用户设置的left和top不起作用。

标记为absolute的标签，会找到第一个position不为static的父级标签作为他的offset parent，然后以offset parent的位置作为自己的唯一的定位参照点，此情况下不会考虑其他兄弟节点。值得注意的是，当offset parent的位置动态的发生了变化之后，标记为absolute的节点不会重新调整自己的位置，这算是absolute的一个劣势吧。

relative和absolute的唯一的区别就在于，标记为relative的标签会以第一个父节点作为offset parent，而需要向上查找。

标记为fixed的标签，会以当前document窗口作为自己的offset parent。

下面来具体看几个例子：

以id为outer，middle和inner的3个div节点为例。

以下两个例子中outer（红色）和middle（黄色）的position属性固定为relative和static，而inner（绿色）在两个例子中分别为absolute和relative。inner的top和left属性值都设置为10px（middle同级还有一个height为10px的兄弟节点，这样middle的位置会向下10px）。

inner为relative的情况下，middle成为inner的offset parent。inner相对于outer的位移为10px,20px.

![relative](/assets/post_resources/css-position/css-position-1.png)

inner为absolute的情况下，position属性值为static的middle被跳过，outer成为inner的offset parent。inner相对于outer的位移为10px,10px。

![absolute](/assets/post_resources/css-position/css-position-2.png)