---
layout: post
title: 谈谈几种javascript继承模式
---

我们知道Javascript对继承的语言级支持是基于prototype的，但是prototype的支持非常基本，让我们很难写出reusable的code，因此我们常常要自己搭建简单的继承框架以实现代码复用。

最近team里在讨论如何改进现有项目的继承机制，借这个机会就谈一谈我对目前常用的继承机制的看法。
我们现在的code convention模仿 Michael Bostock(D3作者)的一个[proposal](http://bost.ocks.org/mike/chart/) 。
它基本上是如下这个样子：

    function Knight(){
        function module(){}
        module.walk = function(){
            /*knight walk*/
        };
        return module;
    }
    function Ninjia(){
        function module(){}
        module.walk = function(){
            /*ninjia walk*/
        };
        return module;
    }

很显然这段代码没有任何复用。
那么我们是如何改进的呢：

    function Person(){
        function module(){}
        module.walk = function(){
            /*person walk*/
        };
        module.speak = function(){};
        return module;
    }
    var Ninjia = Person();
    Ninjia.walk = function(){/*ninjia walk*/};

这种继承方式有什么问题呢，最显然的是我每次要从`Person`继承出一个新的class，
必然要call一遍`Person`这个function，这么多成员方法每次都被重复定义，这是非常浪费开销的。
另外，如果我想在父类的方法上进行扩展，而不是完全重写，这种继承方式是很难做到的，
基本上只能这么做：
  

    var Ninjia = Person();
    var _superWalk = Ninjia.walk;
    Ninjia.walk = function(){
        _superWalk();
        /*Ninjia walk*/
    };

这种勉强的实现不太优雅。
所以这种继承机制不是我们想要的，于是我们比较怀念其他语言里可以直接用`super`等关键字访问父类，javascript能不能也实现这种机制呢？就像下面这样？
  
    Ninjia.walk = function(){
        this._super();
        /*Ninjia walk*/
    }

答案当然是可以的，下面我们来看看一个比较新颖的做法：
  
    //我们先定义一个空的constructor，称之为ZeroClass
    function ZeroClass(){}
    //然后我们定义一个extend方法，用以从一个父类扩展出子类
    function extend(prop) {
            var _super = ZeroClass.prototype = this.prototype;
            var proto = new ZeroClass();
            for (var name in prop) {
                proto[name] = typeof prop[name] == "function" && 
                    typeof _super[name] == "function"? 
                (function(name, fn) {
                    return function() {
                        var tmp = this._super;
                        this._super = _super[name];
                        var ret = fn.apply(this, arguments);
                        this._super = tmp;
                        return ret;
                    };
                })(name, prop[name]) : prop[name];
            }
            function Subclass(){
                this.init.apply(this,arguments);
            }
            Subclass.prototype = proto;
            Subclass.prototype.constructor = Subclass;
            Subclass.extend = extend;
            return Subclass;
        }
        Base = {
            "prototype":{},
            "extend": extend
        }
  
这段代码看起来比较复杂，需要仔细分析下才能搞清楚内部的机制(其实我已经进行了一些化简让它易懂。。)，不过内部复杂不要紧，能使用方便是我们最想要的。我们来一点点的分析下这个`extend`方法。
  
在`extend`的第一行，首先把`ZeroClass`的`prototype`指向我们的父类，也就是当前类，这个`ZeroClass`是继承的关键，每个子类的`prototype`都是它构造出来的。
  
在`ZeroClass`有了prototype之后，我们创建一个`ZeroClass`的实例对象，这个实例对象有父类的所有方法，因此可以作为子类`SubClass`的prototype了，使用`ZeroClass`而不是直接用父类来创建prototype的目的是省下constructor的开销，毕竟`ZeroClass`里什么都没做。
  
然后开始往子类的prototype上放方法了，如果覆盖了父类的方法，这里返回了一个wrapper function：
  
    function() {
        var tmp = this._super;
        this._super = _super[name];
        var ret = fn.apply(this, arguments);
        this._super = tmp;
        return ret;
    }

在这个wrapper function里，在call真正的子类方法之前会把`this._super`指向父类的同名方法，这样我们在子类方法里就可以用`this._super()`来调用父类的同名方法了。
在调用完成之后这个wrapper function又将`this._super`指向了prototype，这样调用其他子类function的时候也可以同样访问同名父类方法了。
  
最后，把extend方法挂到我们新创建的子类上，子类也就用了扩展能力。
  
    Person = Base.extend({
            init：function(){},
            walk: function(){
                console.log("person walk");
            }
        });
    Ninjia = Person.extend({
            init：function(){},
            walk: function(){
                this._super();
                console.log("in ninjia costume");
            }
        });

试试输出？
如果对这种继承机制想有更深的了解，可以看看jquery作者john resig的[原作]http://ejohn.org/blog/simple-javascript-inheritance/
  
这篇文章到这里基本可以结束了，但是我还想再谈谈javascript本身的prototype继承为什么坑爹，我们为什么要用这么复杂的代码来实现其他语言里很基本的继承。
  
我们知道访问一个javascript对象上的属性的时候，如果找不到这个属性，会自动查找prototype上有没有这个属性。这就是javascript继承机制的基本。所以最简单的继承是这个样子的：
  
    function Person(){}
    Person.prototype.walk = function(){/*person walk*/};
      
    function Ninjia(){}
    Ninjia.prototype = Person.prototype;

这种方法的弊端显而易见，只有一个单prototype，如果再从`Ninjia`往下继承，继承到的还是`Person`的prototype，`Ninjia`只有往prototype上放东西，才能被子类继承到，但是这样又会污染`Person`的prototype，怎么办呢，可以创建一个实例对象作为prototype，我们修改这个实例对象，就不用担心污染prototype链了。
  

    function Person(){}
    Person.prototype = function(){/*person walk*/};
      
    function Ninjia(){}
    Ninjia.prototype = new Person();

这种方法的问题其实上面已经提到了，我们为了创建一个新的prototype，必须创建一个`Person`的对象，constructor的开销实际上是没有意义的，所以上面我们用到了一个`ZeroClass`，这是一个小技巧。把这些便利的小技巧封装起来，就抽取出了上面这个`extend`方法。定义了这个`Base` class之后， 以后每次创建新的class就方便了。