# Python2.x与3.x版本区别

### 概述
> Python的3.0版本，常被称为Python 3000，或简称Py3k。相对于Python的早期版本，这是一个较大的升级。
> 为了不带入过多的累赘，Python 3.0在设计的时候没有考虑向下相容。
> 许多针对早期Python版本设计的程式都无法在Python 3.0上正常执行。
> 为了照顾现有程式，Python 2.6作为一个过渡版本，基本使用了Python 2.x的语法和库，同时考虑了向Python 3.0的迁移，允许使用部分Python 3.0的语法与函数。
> 新的Python程式建议使用Python 3.0版本的语法。
> 除非执行环境无法安装Python 3.0或者程式本身使用了不支援Python 3.0的第三方库。目前不支援Python 3.0的第三方库有Twisted, py2exe, PIL等。
> 大多数第三方库都正在努力地相容Python 3.0版本。即使无法立即使用Python 3.0，也建议编写相容Python 3.0版本的程式，然后使用Python 2.6, Python 2.7来执行。

### Python 3.0的变化主要在以下几个方面:
- print 函数
    - Python 2：print语句，语句就意味着可以直接跟要打印的东西，如果后面接的是一个元组对象，直接打印
    - Python 3：print函数，函数就以为这必须要加上括号才能调用，如果接元组对象，可以接收多个位置参数，并可以打印
    - 如果希望在 Python2 中 把 print 当函数使用，那么可以导入 future 模块 中的 print_function
    - print语句没有了，取而代之的是print()函数。 Python 2.6与Python 2.7部分地支持这种形式的print语法。

- 输入函数
    - Python 2：input_raw()
    - Python 3：input()

- True和False
    - Python 2：True 和 False 在 Python2 中是两个全局变量，可以为其赋值或者进行别的操作，初始数值分别为1和0，虽然修改是违背了python设计的原则，但是确实可以更改
    - Python 3：修正了这个变量，让True或False不可变    

- 字符串编码问题
    - 区别
        - Python 3默认使用的是str(unicode)类型对字符串编码，默认使用bytes操作二进制数据流，两者不能混淆！！
            > python3的str类型为unicode类型
        - Python 3有两种表示字符序列的类型：bytes和str。前者的实例包含原始的8位值，后者的实例包含Unicode字符。
        - Python 2也有两种表示字符序列的类型，分别叫做str和Unicode，与Python3不同的是，str实例包含原始的8位值；而unicode的实例，则包含Unicode字符。

        - str对象和bytes对象可以使用.encode() (str -> bytes) or .decode() (bytes -> str)方法相互转化。
        ```python 
        >>> s = b.decode() 
        >>> s 
        'china' 
        >>> b1 = s.encode() 
        >>> b1 
        b'china' 
        ```
        - Python 2：默认编码ascii
        - Python 3：默认编码utf-8

    - Python 3 str(unicode)类型 *(控制台输出和存入数据库都不是正常编码形式了!)*
    
        > 现在， 在 Python 3，我们最终有了 Unicode (utf-8) 字符串，以及一个字节类：byte 和 bytearrays。由于 Python3.X 源码文件默认使用utf-8编码，这就使得以下代码是合法的：
        
        **神奇的变量名可以用中文，不过！~~千万别这样做！**
        
        ```python
        >>>中国 = 'china' 
        >>>print(中国) 
        china
        ```
    
    - 关于base64的调用变化
        - base64需要使用byte类型了，因为Python 2中使用的是str
        - 使用例子：
        
        ```python
        import base64
        a = '中文Abc123'
        _a = base64.b64encode(a.encode())
        print (_a)
        
        #做了base64加密后出来的串是btype型
        >>> b'5Lit5paHQWJjMTIz'

        b = base64.b64decode(_a)
        
        #如果不做decode，那么就是byte型
        print (b.decode())
        >>> 中文Abc123

        ```
        
    - 建议使用format替换%方式的字符串工具
    
        ```python
        fstr = 'COPY {tab}{cols} FROM STDIN {opts}'.format(
                tab='ttt', cols='ccc', opts='ooo')
        ```
        
- 除法运算

    - Python中的除法较其它语言显得非常高端，有套很复杂的规则。Python中的除法有两个运算符
        > /和//
    - 首先来说/除法:在python 2.x中/除法就跟我们熟悉的大多数语言，比如Java啊C啊差不多，整数相除的结果是一个整数，把小数部分完全忽略掉，浮点数除法会保留小数点的部分得到一个浮点数的结果。
    - 在python 3.x中/除法不再这么做了，对于整数之间的相除，结果也会是浮点数。
        > Python 2:	
        
        ```python
        >>> 1 / 2
        0
        >>> 1.0 / 2.0
        0.5
        ```
        > Python 3:
                
        ```python
        >>> 1/2
        0.5
        ```
    - 而对于//除法，这种除法叫做floor除法，会对除法的结果自动进行一个floor操作，在python 2.x和python 3.x中是一致的。
        > Python 2:
        
        ```python
        >>> -1 // 2
        -1
        ```
        > Python 3:
        ```python
        >>> -1 // 2
        -1
        ```
        *注意的是并不是舍弃小数部分，而是执行floor操作，如果要截取小数部分，那么需要使用math模块的trunc函数*
        > Python 3:
        
        ```python
        >>> import math
        >>> math.trunc(1 / 2)
        0
        >>> math.trunc(-1 / 2)
        0
        ```
- 异常
    - 在 Python 3 中处理异常也轻微的改变了
        > 在 Python 3 中我们现在使用 as 作为关键词。        
        捕获异常的语法由 except exc, var 改为 except exc as var。

    - 使用语法except (exc1, exc2) as var可以同时捕获多种类别的异常。 Python 2.6已经支持这两种语法。
    - 在Python 2时代，所有类型的对象都是可以被直接抛出的，在Python 3.x时代，只有继承自BaseException的对象才可以被抛出。
    - Python 2 raise语句使用逗号将抛出对象类型和参数分开，Python 3.x取消了这种奇葩的写法，直接调用构造函数抛出对象即可。
    - 在Python 2时代，异常在代码中除了表示程序错误，还经常做一些普通控制结构应该做的事情
    - 在Python 3中可以看出，设计者让异常变的更加专一，只有在错误发生的情况才能去用异常捕获语句来处理。*（不要使用异常控制逻辑结构！）*

- xrange
    - 在 Python 2 中 xrange() 创建迭代对象的用法是非常流行的。
        > 比如： for 循环或者是列表/集合/字典推导式。
        这个表现十分像生成器（比如。"惰性求值"）。但是这个 xrange-iterable 是无穷的，意味着你可以无限遍历。
        由于它的惰性求值，如果你不得仅仅不遍历它一次，xrange() 函数 比 range() 更快（比如 for 循环）。尽管如此，对比迭代一次，不建议你重复迭代多次，因为生成器每次都从头开始。
    
    - 在 Python 3 中，range() 是像 xrange() 那样实现以至于一个专门的 xrange() 函数都不再存在。

- 八进制字面量表示
    - 八进制数必须写成0o777，原来的形式0777不能用了；二进制必须写成0b111。
    - 新增了一个bin()函数用于将一个整数转换成二进制字串。 Python 2.6已经支持这两种语法。
    - 在Python 3中，表示八进制字面量的方式只有一种，就是0o1000。

- 不等运算符
     - Python 2中不等于有两种写法 != 和 <>
     - Python 3中去掉了<>, 只有!=一种写法 *(还好，我从来没有使用<>的习惯)*

- 去掉了repr表达式``
    - Python 2 中反引号``相当于repr函数的作用
    - Python 3 中去掉了``这种写法，只允许使用repr函数。*(还好，我还不知道有反引号这种用法)*

- 多个模块被改名（根据PEP8）

        旧的名字| 新的名字
        ---|---
        _winreg |	winreg
        ConfigParser |	configparser
        copy_reg |	copyreg
        Queue |	queue
        SocketServer |	socketserver
        repr | reprlib

- StringIO模块现在被合并到新的io模组内。 
- new, md5, gopherlib等模块被删除。 
- httplib, BaseHTTPServer, CGIHTTPServer, SimpleHTTPServer, Cookie, cookielib被合并到http包内。
- 取消了exec语句，只剩下exec()函数。 

- 数据类型
    - Python 3去除了long类型，现在只有一种整型——int，但它的行为就像Python 2版本的long
    - 新增了bytes类型，对应于Python 2版本的八位串，定义一个bytes字面量的方法如下：
    
    ```python
    >>> b = b'china' 
    >>> type(b) 
    <type 'bytes'> 
    ```

- import文件方式 
    - [*(python3-cookbook来源参考)*](http://python3-cookbook.readthedocs.io/zh_CN/latest/c10/p03_import_submodules_by_relative_names.html)
    - [*(相对导入和绝对导入参考)*](http://kuanghy.github.io/2016/07/21/python-import-relative-and-absolute) 
    - Python 2 缺省为相对路径导入，Python 3 缺省为绝对路径导入
    - 如果是在某个模块中直接跑，需要导入这个模块上一级目录，那么需要在这个模块的运行py文件顶部加入下面代码，标识把上一级的导入到syspath搜索路径*（Python 3 缺省是绝对引用）*：

    ```python
        import sys
        sys.path.append("..")
    ```
    - 使用包的相对导入，使一个模块导入同一个包的另一个模块
    - 举个例子，假设在你的文件系统上有mypackage包，组织如下：
    
    ```python
        mypackage/
            __init__.py
            A/
                __init__.py
                spam.py
                grok.py
            B/
                __init__.py
                bar.py
    ```
    
    - 如果模块mypackage.A.spam要导入同目录下的模块grok，它应该包括的import语句如下：
    
    ```python
        # mypackage/A/spam.py
        from . import grok
    ```
    - 如果模块mypackage.A.spam要导入不同目录下的模块B.bar，它应该使用的import语句如下：
    ```python
        # mypackage/A/spam.py
        from ..B import bar
    ```
    
    - 在包内，既可以使用相对路径也可以使用绝对路径来导入。 举个例子：
    ```python
        # mypackage/A/spam.py
        from mypackage.A import grok # OK
        from . import grok # OK
        import grok # Error (not found)
    ```

- 函数声明的新特性 *(原文地址:[Python3新特性——声明参数类型的函数](http://hgoldfish.com/blogs/article/83/))*
    
    > Python 的函数参数不需要指定类型，这种灵活又智能的设定让程序员们写代码时不用花时间去考虑定义变量的类型，又方便程序员设计出灵活的程序。写过Python代码的人朋友们一般会觉得写Python程序很爽。
    
    > 不过，因为在定义函数的时候也没有参数的传递，合作者往往难以确定传入的参数类型。再者，即使IDE可以自动地推定变量的类型，也难以推断函数参数的类型

    **Python 3 已经克服了这个缺点，可以声明参数类型**
    
    ```python
    import io
    
    def add(x:int, y:int) -> int:
        return x + y
    
    def write(f: io.BytesIO, data: bytes) -> bool:
        try:
            f.write(data)
        except IOError:
            return False
        else:
            return True
    ```
    
    > 在这函数里面，同时声明了两个函数参数的类型以及add()函数的返回值类型。可以看出，声明一个函数参数的类型，只要在参数名称的后面加个":"号，带上类型名称就行了。声明函数的返回值类型，只要在函数声明结束之前，也就是":"号之前加入一个"->"，带上类型名称。
    我想，当调用者看到这个函数签名的话，一定能够很快地知道他需要传入哪种类型的参数了。IDE也可以通过函数签名来推定参数的类型。
    即使Python能够声明函数的参数，它仍然是一门动态语言的。与C++、Java等静态语言不同，声明参数的类型并不是强制的。可以单独对函数的任何一个参数声明类型。也可以像以前的Python程序那样，完全不声明类型。
    实际上，写在":"号后面的并一定是一个类型。Python把这种写法称为"annotations"(标注)，在运行的时候完全不使用它。它是专门设计出来给程序员和自动处理程序看的。任何可被计算出来的东西都可以写在那里。
    

- 其他
    - dict的.keys()、.items和.values()方法返回迭代器，而之前的iterkeys()等函数都被废弃。
    - 同时去掉的还有 dict.has_key()，用 in替代它吧 。
    - dict的keys，Python 2中返回的是一个list，因此可以调用list的sort排序，Python 3 中需要使用sortd函数对dict出来的keys排序后返回一个list
    
        ```python
        a = {'2':'aa','1':'bb','7':'ad'}
        b = a.keys()
        c = sorted(b)
        print (c)
        >>>['1', '2', '7']
        ```
    - class中的函数加两个下划线，自动变为私有函数，外部通过class的实例是无法访问的
        ```python
        class A(object):
            def __pri_func():
                pass
        
        ```
        
    - __pycache__文件夹
        - 用python编写好一个工程，在第一次运行后，总会发现工程根目录下生成了一个__pycache__文件夹，里面是和py文件同名的各种 *.pyc 或者 *.pyo 文件。
        - 这样工程较大时就可以大大缩短项目运行前的准备时间
        - 该文件夹出现在Python3.2及其后的版本中，Python2下的编译文件和源文件放同目录。
    








