# python学习笔记
## 1 python模块
一个.py文件就是一个python模块。使用import引入模块，from...import...从某模块中导入某个部分（eg.函数）。
### 1.1 import
Python 的首次 import 会 “执行所有的顶层代码” ，**甚至可以做所有运行时能够做到的事**。
### 1.2 import搜索路径

当你导入一个模块，Python 解析器对模块位置的搜索顺序是：

-   1、当前目录
-   2、如果不在当前目录，Python 则搜索在 shell 变量 PYTHONPATH 下的每个目录。
-   3、如果都找不到，Python会查看默认路径。UNIX下，默认路径一般为/usr/local/lib/python/。

模块搜索路径存储在 system 模块的 sys.path 变量中。变量里包含当前目录，PYTHONPATH和由安装过程决定的默认目录。
### 1.3 命名空间和作用域
一个 Python 表达式可以访问局部命名空间和全局命名空间里的变量。如果一个局部变量和一个全局变量重名，则局部变量会覆盖全局变量。
```python
#!/usr/bin/python
# -*- coding: UTF-8 -*-
 
Money = 2000
def AddMoney():
   # 想改正代码就取消以下注释:
   # global Money
   Money = Money + 1
 
print Money
AddMoney()
print Money

```
### 1.4 python import时和运行时的关系
[知识链接](https://halfclock.github.io/2019/06/07/python-import-and-running/)
* import时是指在python命令行中或者.py脚本语言中import模块，eg. import test.py
* 运行时是指在终端命令行中直接执行.py脚本文件，eg. python test.py         

直接说结论：**第一次 import 时与运行时的行为几乎一致，都会按照 import 的顺序从上至下执行 .py 脚本文件的顶层代码（顶层代码指 .py 脚本文件中的所有非函数方法的定义体、被其他顶层代码调用的函数方法的定义体和即使未被调用的装饰函数的定义体），其他的所有函数只会被解释器编译，不会执行函数定义体。区别有三点：**
1. **import 时只有在第一次 import 时才会存入 `__pycache__` 的 .pyc 文件并执行，之后的 import 时都已经在`__pucache__.pyc`中。**
2. **python 导入时与运行时的重大差别在于 `if __name__ == '__main__':`，import 时`__name__`为模块名，运行时`__name__`为`__main__`。**
3. **import 时发生在 python 命令行中，在这里 python 程序在 import 完 .py 后还没有结束，不会触发垃圾回收机制，而运行时则会在程序执行完之后触发垃圾回收机制。**
#### 1.4.1 import时
在 python 命令行**首次输入** `import A` 时会按照**代码顺序**执行以下几条：

1.  **会执行模块 A 中的所有顶层代码。**

1.  **会执行模块 A 导入的所有模块的顶层代码。**

1.  **执行的顶层代码中**：

    1.  **若是类的定义体，那么会执行类的定义体、但是只会编译类方法，不会执行类方法定义体。**
    1.  **若是函数，则解释器只会编译此函数，不会执行函数定义体。**

1.  **在执行的类的定义体中**：

    1.  **若是嵌套类**，那么会执行嵌套类的定义体，规则与第三条规则中的类定义体情况一致。
    1.  若该类被装饰、**在执行完类定义体后执行装饰器函数定义体。**
    1.  与该类是否是某个类的子类无关（正常执行）。

1.  **执行装饰器函数定义体中**：

    1.  **若是嵌套函数，那么只会编译此函数，不会执行该函数定义体。**
#### 1.4.2 运行时
运行时的执行规则与导入时基本一致：

1.  在系统命令行调用 `python evaltim.py` 与在 python 命令行调用 `import evaltime` 的**执行顺序和执行规则一致**。
1.  不同的是，运行时会执行 `if __name__ == '__main__':` 代码块的代码。

值得一提的是，如果你将 `if __name__ == '__main__':` 代码块的代码**全部都变成顶层代码**，也就是去掉 `if __name__ == '__main__':` 这一行，**那么运行时的输出与首次导入时的输出几乎一致，只是最后一行销毁 ClassOne 实例的代码不会输出。**

这是因为导入时发生在 python 命令行中，在这里 python 程序在 import 完 evaltime 后还没有结束，不会触发垃圾回收机制。
## 2 python包
[知识链接](https://www.runoob.com/python/python-modules.html)         
包是一个分层次的文件目录结构，它定义了一个由模块及子包，和子包下的子包等组成的 Python 的应用环境。

简单来说，包就是文件夹，但该文件夹下必须存在 \__init\__.py 文件, 该文件的内容可以为空。 \__init\__.py 用于标识当前文件夹是一个包。

考虑一个在 **package_runoob** 目录下的 **runoob1.py、runoob2.py、\__init__.py** 文件，test.py 为测试调用包的代码，目录结构如下：
```
test.py
package_runoob
|-- __init__.py
|-- runoob1.py
|-- runoob2.py
```
### 2.1 \_\_init\_\_.py文件
`__init__.py` 文件在 Python 包中具有初始化包、控制包的导入行为和提供包级别功能的作用。我们可以根据需要在 `__init__.py` 文件中编写代码来满足这些需求，并更好地组织和管理你的包。     
`__init__.py` 文件的主要作用是初始化包。当导入一个包时，Python [解释器](https://so.csdn.net/so/search?q=%E8%A7%A3%E9%87%8A%E5%99%A8&spm=1001.2101.3001.7020)会首先执行该包下的 `__init__.py` 文件。我们可以在 `__init__.py` 文件中执行一些[初始化操作](https://so.csdn.net/so/search?q=%E5%88%9D%E5%A7%8B%E5%8C%96%E6%93%8D%E4%BD%9C&spm=1001.2101.3001.7020)，例如设置包的全局变量、导入特定模块或子包等。
```python
# __init__.py

print("Initializing my_package...")

# 设置包级别的变量
package_variable = 10

# 导入模块或子包
from . import module1
from .subpackage import module2

# 执行其他初始化操作，可以在 __init__.py 文件中执行其他的初始化操作，例如注册插件、加载配置文件、初始化数据库连接等。
# ...
# 注册插件
register_plugin()

# 加载配置文件
load_config()

# 初始化数据库连接
initialize_database()

```
## 3 !/usr/bin/env python
#!/usr/bin/env python 是一个在 Unix-like 操作系统中用于指定脚本解释器的特殊语法。这一行通常被称为"shebang"（井号和叹号的组合，#!），其目的是告诉系统使用指定的解释器来执行脚本。

具体来说，#!/usr/bin/env python 表示使用 env 命令来找到系统中的 python 解释器，并使用它来执行脚本。/usr/bin/env 是一个在Unix-like系统中用于在用户的环境变量中查找可执行文件的工具。这样的写法的好处在于它允许用户在不同的环境中使用不同版本的Python解释器，而不需要指定确切的解释器路径。

例如，如果系统中有Python 2和Python 3两个版本，#!/usr/bin/env python 将会使用环境变量中默认的Python版本。如果要明确指定使用Python 3，可以写成 #!/usr/bin/env python3。

在使用shebang时，需要确保脚本文件具有执行权限。执行权限可以通过 chmod +x script.py 命令添加。然后，用户可以通过 ./script.py 直接运行脚本，而不需要显式地调用Python解释器。

## 4 python的容器数据类型

| 容器 | 有无序(可否数字索引) |可否变|可否重复|不同类型是否兼容|
| --- | --- |---|---|---|
| list()/[] | 有序可索引 |可变|可重复|不同类型兼容|
|tuple()/()|有序可索引|不可变|可重复|不同类型兼容|
|dict()/{:}|无序不可索引|可变|键不可重复|不同类型兼容|
|set()/{}|无序不可索引|可变|不可重复|不同类型兼容|

### 4.1 可变和不可变数据类型
* 可变：值变，址不变，支持原地修改                
* 不可变：值变，址也变，不支持原地修改

|||||
|---|---|---|---|
|可变数据类型|list()/[]|set()/{}|dict()/{:}|
|不可变数据类型|int|string|tuple()/()|
## 5 python装饰器

## 6 django原理
