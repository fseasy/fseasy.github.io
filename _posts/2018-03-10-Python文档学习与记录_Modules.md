---
layout: post
title: Python文档学习与记录——Modules
date: 2018-03-10
categories: 技术 
tags: Python 学习
onewords: module是什么；Package是什么
---
> 本文主要对Python文档`Modules`做一个学习总结、记录。

## 背景

UPDATE: 2019年1月9日，内部做了工程经验分享，对Python Modules看得更多一些，对内容做一些更新。

————

前几天因为`import`相关的问题导致代码没有通过公司的编码规范检查，因为任务优先，当时就选择临时豁免了。
今天算是实现当时的想法，对[6. Modules](https://docs.python.org/2/tutorial/modules.html)做一个学习和记录。
此外，从[How To Package Your Python Code](http://python-packaging.readthedocs.io/en/latest/index.html)学习了package的结构，下一个工程就全面按照这个规范来做，这样就总算是“代码规范”了吧~


## 正文

本文主要由以下内容组成， 

1. Module相关：Module是什么；module相关的知识
2. Package相关：Package是什么；import相关的知识；如何去组织package；开发实践（开发Package时怎么才能方便测试？）

### Module

#### Module是什么

原文的定义是：

> A module is a file containing Python definitions and statements.

就是把Python的定义、声明给放到一起的文件。一个`*.py`就是一个Module，module name就是文件名。Module本身有一个`__name__`的变量，一般存储的就是自己的名字。

Module应该是Python代码组织结构的一环：Package - Module - Class。 Module也是代码逻辑组织与文件组织共同体现。
写Module的意义，当然是为了重用性和可维护。公用功能拆分、大逻辑解耦，都需要落实到module上。

Module里可以包含函数、变量的定义，也可以包含一些可运行的代码，`These statements are intended to initialize the module`.

一个Module包含自己独立的符号表(private symbol table)，module内部声明的变量、函数等都在这个符号表里（import进来的也放在这里），类似C++的命名空间。这使得我们不必担心不同module间重名的问题。

#### 将Module作为Script运行

Script通常指可以运行的脚本，如果一个module可以通过Python解释器直接运行（即 `python xx.py` ），那么我们称被运行的module为script. 区别module与script，是根据它的运行状态：被直接运行的是script，否则就是module.

而之所以要区分这个概念，是因为当一个module成为script后，其 `__name__` 属性值发生了变化：由原来的自身名字，变为了 `__main__`. 这个行为对程序流、relative import时路径搜索有关键影响。

例如，我们常常在一个module里写上

    if __name__ == "__main__":
        # some statements for running as scripts

这就是区分其作为script或被import的module时不同行为的：只有作为script被直接运行， `if` 下面的代码才会被解释执行。


#### `pyc`和`pyo`相关

这两个都是源程序编译为的机器无关的字节码，跟Java的class一样，是可以独立于操作系统、机器结构等的——只要有对应的interpreter, 都可以正常解释为机器码。

`pyo`是使用`python -O xxx.py`时，被依赖的module被编译成的文件格式，也就是做了优化的格式。相比`pyc`，

>  The optimizer currently doesn’t help much; it only removes assert statements. When -O is used, all bytecode is optimized

官网上面的说法感觉有点矛盾……这里姑且取第一句，认为`pyo`就是把`assert`的语句给删了，仅此而已。除了`-O`, 还有`-OO`的优化，似乎只是把`_doc__`给删了……为了简单，后面就只说`pyc`, 不再提`pyo`.

`pyc`已经是字节码，因此已经可以直接发布了——不再需要源文件。当你的代码不开源时，一定程度上可以用来反工程（`This can be used to distribute a library of Python code in a form that is moderately hard to reverse engineer.`）。根据我之前在某动实习时师兄告诉我的经验，pyc是挡不住的，需要先把python源代码利用Cython给转成C代码，再拿C编译器编译成so，再发布……

当既有`pyc`, 又有`py`是，解释器首先看`pyc`是不是最新的，是就直接用，不是再编。关键是怎么看是不是最新的呢？之前我以为就是比两个文件的时间戳，不过真实的是：

> The modification time of the version of spam.py used to create spam.pyc is recorded in spam.pyc, and the .pyc file is ignored if these don’t match.

把源文档变更时间直接写到pyc里了，的确更加稳妥！

最后，`pyc`只是加速启动时间，跟运行速度无关。

#### `dir`函数和`__builtin__`和`sys`的特殊变量

`dir`是内建函数，可以把Module内的全局变量（module的符号表）返回为list；特别地，如果是在交互式环境直接使用（不带参数），那么返回的是整个交换式环境下的所有名字，变量啊、类啊都混在了一起。而其中就有一个`__builtins__`的特殊名字，是所有内建名字的代替吧。文档里说用`import __builtin__; dir()`可以看到一大串，可以看到包括Errors定义、内建函数、特殊变量等。

最后再说`sys`, 先说`sys.path`, 决定了import搜索路径的东西。之前都是通过操作这个列表来完成不同层级的module的导入的。有一个初始化顺序需要了解一下：


    the directory containing the input script (or the current directory).
    `PYTHONPATH` (a list of directory names, with the same syntax as the shell variable PATH).
    the installation-dependent default.


此外还发现两个有意思的，`sys.ps1`, `sys.ps2`， 第一个是交互式解释器第一层的提示字符串(prompts), 第二个是第二层；默认情况下是`>>>`和`...`；是可以直接赋值的。

### Package

#### Package是什么

Package是Module的集合，`a way of structuring Python’s module namespace by using “dotted module names”`. 直观而不严谨的说明，一个package通常包含一个层次化的module集合，对应到文件系统，就是层次化的文件结构。文件系统的层次化用`/`表示，对应到Package，就是`.`. 例如文件系统结构 `a/b`, 对应的mudule表示是`a.b`. 特别地，为了防止意外的引入不期望的文件，一个文件夹要对应到一个Package或Sub-Package，必须包含一个`__init__.py`文件。

需要注意的是，Package只是Python用来组织module的concept, 仅仅对应到逻辑上的概念。在Python实际运行中真实存在的对象，还是是module，具体的，Package文件夹下的`__init__.py`就是Package在运行中的表示。可以试试在`__init__.py`中打印`__name__`，其值就是Package的名字。

#### `__init__.py`中的内容

`__init__.py`可以是一个空文件，仅仅表示这个文件夹对应到一个Package;不过，它也可以作为这个package初始化代码存放的地方。`__init__.py`中定义的名字是被放在该package的符号表中的。如果一个package里有sub-package, 只有把这个sub-package一import的方式放到`__init__.py`中，才能通过package名字访问到该sub-package。例如：


    package-store-dir
        setup.py
        package
            __init__.py
            sub_package.py
                __init__.py


必须在`package-store-dir/package/__init__.py`里写上`import package.sub_package as sub_package`之后，才能在外部的Scripts中这样访问sub-package: `import package; package.sub_package`, 否则在package上是看不到sub_package的！当然，可以这样做： `import package.sub_package`，这是直接通过path引入sub-package，而不是通过package符号表的方式去索引。这大概也是非常容易混淆的点。

此外，它还有一个特别的变量`__all__`，它是一个list，只在`from package import *`这种语法出现的时候有意义——相当于在`import *`时人为覆盖了由Python自己枚举的`__all__`. 特别的，Python自己枚举的`__all__`也不是全部的module，而是a. 在`__init__.py`里定义的names； b. 在之前已经显式import进来的属于该package的子模块。不知道自己理解（翻译）对不对，或者说又没有解释清楚——可以参考[import * from a package](https://docs.python.org/2/tutorial/modules.html#importing-from-a-package). 在生产代码中，永远不要使用`import *`，它是代码可读性降低，同时可能让你implicitly覆盖掉import进来的名字。


#### Absolute import, relative import(Explicit, Implicit)

import的方式分为两种，分别是绝对引入(absolute import)和相对引入(relative import)。在Python2中相对引入还分为 Explicit relative import 和 Implicit relative import, 而在Python3中，Implicit relative import 已经被废弃。

所谓绝对与相对引入，可以类比文件系统中的绝对和相对路径。只不过，文件系统的绝对路径是唯一的（也就是从`/`开始），但是绝对引入确实基于一个TOP-LEVEL的路径列表来算的：凡是在这个TOP-LEVEL中的任意一个路径下找到了需要import的包，那么这就是绝对引入。

> TOP-LEVEL, 其实可以等价`sys.path`. 按照先后顺序，它由以下内容构成： 1. 入口script所在文件夹的绝对路径 2. PYTHONPATH环境变量 3. standard modules位置 4. site.py（在standard modules中）引入的site-packages, 和其中`*.pth`文件引入的扩展位置.

当我们禁止implicit relative import后，其实absolute import 和 explicit relative import非常好区分：explicit relative import 的import路径，总是以 `.` 开头的，且语法只能是`from IMPORT-PATH import NAME ...`, 例如


    from . import input_name
    from .input_name import InputName
    from ...util import text
    # import .input => ERROR

不过如果我们允许了implicit relative import, 那么我们从代码上格式就没法区分其与absolute import的区别——只有实际看路径是相对路径，还是绝对路径，才知道是何种import. 

所以，为了不让人混淆，最好就是禁止implicit relative import. 在Python2.7中，可以通过 `from __future__ import absolute_import`来实现这个行为。

为了更直观地理解上面说的，复制官网的例子：


    sound/                        Top-level package
        __init__.py               Initialize the sound package
        formats/                  Subpackage for file format conversions
                __init__.py
                wavread.py
                ...
        effects/                  Subpackage for sound effects
                __init__.py
                echo.py
                surround.py
                ...


假设`surround.py`要引入`echo.py`，

1. 如果是 `import sound.effects.echo` 显然是绝对引入
2. 如果是 `import echo`，那么是隐式相对引入
3. 如果是 `from . import echo`，那么是显式相对引入。

#### 为何最好禁止implicit relative import

在Python2.7上禁止implicit relative import 基本是共识。原因是这种方式功能不完备，极易带来混淆, 甚至导致不可预期的行为。

首先说功能不完善：还是上面的例子，现在假设 `sourround.py`要引入`wavread.py`, 那么

1. 绝对引入： `import sound.formats.wavread`
2. 隐式相对引入： 不可能
3. 显式相对引入： `from ..formats import wavread`

可以看到绝对引入和显式相对引入是功能完善的，而隐式相对引入，实际是很弱，仅仅是特定情况下的一种便利！这种便利其实完全可以通过显式相对引入来达到——基本可以认为多加一个`from .`就行了。

“混淆”体现在，除了让人从思想上难以分清绝对和相对引入，还会在module存在重名的情况下带来事实上的混淆。例如：


    main.py
    foo/
        __init__.py
        bar.py
        util.py
    util.py


`main.py`如入口script, 而在bar.py中写有`import util`, 可以看到在其同级，和`main.py`的同级——也就是TOP-LEVEL，都有`util.py`, 
那么这个时候， `bar.py`里究竟引入的是哪一个`util.py`呢？如果你不深入了解import在未禁止implicit relative import条件下
的路径搜索逻辑，那么你只有猜测。反过来，如果你禁止了隐式相对引入，同时要么用`import util`就必然是TOP-LEVEL下的`util.py`, 
而要用同级的`util`, 就使用`from . import util`即可！

最后，解释一下其为何会导致“不可预期的行为”.


被直接运行的，如前面所说，是`Scripts`, 也就是有`__name__ == "__main__"`的Module. 

应该可以说，绝对引入(absolute import)，相对引入(relative import, including implicit relative import, explicit relative import) 都是在Package下的概念！对于`Scripts`是不存在的，Scripts只有一个 search path的概念：从search path从去找package名字，找到就导入。

上面的说法估计不是正确的，但是Scripts没有 implicit relative import是必然的。因为implicit relative import 依赖的是当前`module.__name__`的层次化表示的名字，而Scripts的`__name__ == "__main__"`，没有层次化名字！ 

前面说了半天，还没有说什么是绝对引入和相对引入。

#### `from __future__ import absolute_import`以及正确使用

这里参考的是[python __future__ package的几个特性](http://www.cnblogs.com/harrychinese/p/python_future_package.html)，我觉得写得很棒！

虽然名字是`absolute_import`，但是其真正含义是**禁止隐式相对引入**.

用了`from __future__ import absolute_import`的Module, 如果把这个Module当作Scripts来运行，常常报这个错： `ValueError: Attempted relative import in non-package`. 原因就是前面说到的，相对引入的概念只存在于Package下（依赖于`module.__name__`来路由位置），Scripts是没有这个东西的。

**所以，一定要清楚，你写的Module是Package的一部分，还是用来作为Scripts运行的**。二者不要想得兼。自己以前总是混写，以后得规范了。

推荐方法如ref的文章所言，Package的Module内用绝对引入或显式相对引入，内部定义一个main函数，然后再写一个scripts调用。当然，这个main放在scripts里也不错。


#### import一个Module发生了什么

假设要import的module名称是module_a, 根据[the import statement](https://docs.python.org/2/reference/simple_stmts.html#the-import-statement), 引入这个module时的大致行为如下：

1. 检查`sys.modules`是否包含`module_a`的名字
    a. 如果包含，则直接返回查找的`instance-of-module_a`；
    b. 否则，对`module_a`做初始化操作，并在`sys.modules`中创建 `key= name-of-module_a`, `value = instance-of-module_a`
2. 在本地命名空间创建一个名字（具体名字根据import语法确定），将该名字绑定到`instance-of-module_a`


#### 如何去组织Package

来自[How To Package Your Python Code](http://python-packaging.readthedocs.io/en/latest/index.html), 不赘述。

此外，[Building and Distributing Packages with Setuptools](http://setuptools.readthedocs.io/en/latest/setuptools.html)是更加详细的setup教程。

#### 开发实践

整体思路是，应该按照Packge的组织去写。然后不嫌麻烦的话，可以把Scripts单独放到一个文件夹下；如果嫌麻烦，可以把Scripts直接放到相应的sub-package下，因为是package，自然是从包的位置开始去引入，因此这个scripts的位置不影响其import操作，所以最后调试好了，再把这些Scripts放到一个bin文件夹里。

待实践后确定……