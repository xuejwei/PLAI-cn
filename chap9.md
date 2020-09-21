# 9 递归和循环：子程序与数据

**递归**指的是自我引用的行为。编程语言中存在（至少）两种形式的递归：数据的递归和控制的递归（程序行为，也就是函数的递归）。

## 9.1 递归与循环数据

数据中的递归还可以分两种情况：引用与自身相同**类型**的事物，或者就是直接引用**自身**。

第一种情况即我们传统称为**递归数据**。例如，树是一种递归数据结构：任一节点可以有若干子节点；每个子节点自身也是树。不过，如果编写程序遍历树，无需记录哪些节点已经被访问。树是有穷的数据结构。

与之对应的是图这种**循环**（cyclic）数据：节点引用其他节点，可能最终通过引用链引用回自身。（当然，节点还可以直接引用自身。）遍历图的时候，如果不记录已访问过的节点，计算过程就可能**发散**，即不会终止。图的算法需要记住已访问过的节点，从而避免重复遍历。

给我们的语言添加递归的数据结构，如链表或树，比较简单直接。主要需要实现两点：

1. 创建复合结构（compound structure）的能力（例如节点可以引用子树）；
2. 结束递归的能力（如树结构的叶节点）

**练习**
> 给语言添加内建数据类型：链表、二叉树

添加循环数据更为微妙。考虑循环数据的最简单形式，指向自身的单元格：
    
<center><img src="./imgs/pict.png" /></center>

试试在Racket中定义它。尝试：

```Racket
(let ([b b])
  b)
```

但这行不通：let中右边那个`b`未绑定。把语法糖解开可以看的更清楚：

```Racket
((lambda (b)
   b)
  b)
```

为了清楚起见，我们可以重命名函数中的`b`：

```Racket
((lambda (x)
  x)
 b)
```

明显`b`未绑定。

不使用额外的Racket构造的情况下【注释】，显然我们无法直接创建循环数据。我们需要给数据创建“地址”，然后在该地址中引用自己。注意这里是用了“然后”，它暗示时间的概念，即我们需要使用赋值操作。这样的话，我们可以试试用`box`来实现。

> 指`shared`构造，不过其他语言基本上都没有这个机制，所以这里我们也不深究它了。我们这里学习的东西正是`shared`幕后实现的基本原理。

计划如下：首先，创建box并将其绑定到某个标识符，设为`b`；然后改变box中的值，我们希望其中存什么呢？当然是对自身的引用。怎么获得该引用呢？通过名字`b`。通过这种方式，我们创建了环状数据：

```Racket
(let ([b (box 'dummy)])
  (begin
    (set-box! b b)
    b))
```

注意，上面的程序在Typed PLAI中**无法**运行，后面会谈到如何给该程序添加类型。现在，要运行上面的程序，请使用动态类型的（`#lang plai`）语言。

运行上面的程序，Racket显示`#0=’#&#0#`。这个表达式正是我们想要的。回想一下，前面提到过`#&`是Racket中box的显示方式。`#0=`（其中0换成其他数也是一样）是Racket中对于循环数据的命名方式。因此，上面结果字面意思就是“`#0`被绑定到了一个box，其内容为`#0#`，即绑定到`#0`的东西，即它自己”。

**练习**
> 在你自己的解释器中运行与这段代码，确保其产生**循环的**数据值。怎么检测这一点呢？

上述思想可以用于其它数据类型。通过这种方式，我们能够创建循环的链表、图等等。核心思想就是分两步做：先命名一个空的占位符；然后修改占位符中的内容为其自身；要获取“自身”，使用第一步中绑定的名字即可。当然，不限于“自循环”：我们也可以创建相互循环的数据（没有某个元素是循环的，但它们的组合是循环的）。

## 9.2 递归函数

澄清一下名词，递归函数不是引用与自身相同**类型**的函数，而是引用**自身**的函数。首先我们的语言需要已经添加了条件分支的特性（比如说前面[第五章](./chap5.md)中，添加了条件指令判断是否是0），这样我们才能写出有意思的程序。

首先，用递归实现阶乘函数：

```Racket
(let ([fact (lambda (n)
              (if0 n
                   1
                   (* n (fact (- n 1)))))])
  (fact 10))
```

这根本行不通！它将报错内层的`fact`未绑定，和前面循环数据的例子相同。

出现这种错误我们并不感到奇怪。毕竟到目前为止，我们实现的绑定机制并不会自动使函数定义支持循环（事实上，在一些早期的编程语言中，函数也不自动支持循环：递归被当作特殊的**特性**）。想要递归的话——即某个函数定义可以循环的引用自己——我们必须手工实现这点。

> 如果按惯例在**顶层**定义函数，你就不会遇到问题。顶层的绑定意味着它要么是变量，要么是box。所以下面说的模式基本上自动就帮你完成了。这也是为什么当你需要局部循环引用的时候，必须使用`letrec`或者`local`，而不是`let`的原因。

那么解决手段也很明了：问题和上一个类似，方案就也用一样的。还是三步走：先创建占位符，然后在需要循环引用的地方使用该占位符，最后在使用之前要对占位符赋值：

```Racket
(let ([fact (box 'dummy)])
  (let ([fact-fun
         (lambda (n)
           (if (zero? n)
               1
               (* n ((unbox fact) (- n 1)))))])
    (begin
      (set-box! fact fact-fun)
      ((unbox fact) 10))))
```

事实上，我们并不需要`fact-fun`：这样写只是为了清晰起见。注意到`fact-fun`不是递归的，而且可以认为它是标识符而不是变量，所以我们可以直接使用它的值：

```Racket
(let ([fact (box 'dummy)])
  (begin
    (set-box! fact
              (lambda (n)
                (if (zero? n)
                    1
                    (* n ((unbox fact) (- n 1))))))
    ((unbox fact) 10)))
```

这里有点小瑕疵，我们使用`fact`的时候总得`unbox`。如果语言中有变量，这么实现看起来更完美：

```Racket
(let ([fact 'dummy])
    (begin
      (set! fact
            (lambda (n)
              (if (zero? n)
                  1
                  (* n (fact (- n 1))))))
      (fact 10)))
```

> 事实上，变量的一个用途就是简化上述模式的去语法糖过程，不再需要每次使用循环绑定的标识符时都得unbox。另一方面，通过一些额外的努力，去语法糖过程也可以把unbox带掉。

## 9.3 草率的观察

到这里我们发现一个遵从同样时间顺序的模式：创建、更新、使用。我们可以将这个过程裹在语法糖中。考虑实现下面的语法：

```Racket
(rec name value body)
```

举个例子：

```Racket
(rec fact
     (lambda (n) (if (= n 0) 1 (* n (fact (- n 1)))))
     (fact 10))
```

它将计算得到10的阶乘。该语法糖解开会得到：

```Racket
(let ([name (box 'dummy)])
  (begin
    (set-box! name value)
     body))
```

这里，我们假设`value`和`body`中所有对`name`的引用都被改写为`(unbox name)`；或者换种方法，我们也可以使用变量：

```Racket
(let ([name 'dummy])
  (begin
    (set! name value)
    body))
```

这自然就导致一个问题：如果我们搞砸了顺序呢？最有意思的是，如果我们在更新`name`到实际值之前使用它呢？那么我们将看到初始化时系统给该结构的无意义值，也就是原始形式的占位符。

最简单可以描述此问题的例子是：

```Racket
(letrec ([x x])
  x)
```

或者等价的：

```Racket
(local ([define x x])
  x)
```

在大多数Racket变体中，这会泄露占位符的初始值——这个值并没打算给大家使用。麻烦的地方是，这又是个合法的值，这意味着它至少可以被用于一些计算中。然而，如果无意中访问和使用它，那么后续的计算就是瞎扯。

这个问题通常有三种解决方案：

1. 确保该值足够模糊，以至于无法在有意义的上下文中使用该值。这意味着像`0`这种值就不能用，事实上语言中绝大多数数据类型都不该用。取而代之，语言应该创建一种新类型的值专作此用。将该值传入其它任何操作都将导致错误的抛出。
2.
对于任意一处标识符的使用，明确地检查其值是否是这个特殊的“过早”值。虽然这在技术上是可行的，但它会对程序造成了巨大的性能损失。因此，通常只有教学语言这么做。
3. 只允许递归构造用于绑定函数中，而且要求该绑定的右项必须**在语法上**是函数。不幸的是，这个解决方案过于激进，比如说它不允许了我们写出图这样的结构。

## 9.4 不用到显式的状态

聪明的你可能想到，还有一种方法可以定义递归函数（递归数据也是一样），而无需用到显式的赋值操作。

**思考题**
> 你应该已经明白，当我们使用`let`来定义递归函数时出了什么问题。请再试试。提示：需要更多的替换。不够再加，加满！

仅使用函数（字面意思上）获得递归是个了不起的想法。Daniel P. Friedman和 Matthias Felleisen在《The Little Schemer》一书中很好的描述了其做法。你可以读一下其[在线样章](http://www.ccs.neu.edu/home/matthias/BTLS/sample.pdf)。

**练习**
> 这个方案中用到了状态吗？有没有间接的用到呢？