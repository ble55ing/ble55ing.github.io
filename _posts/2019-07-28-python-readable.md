---
layout: post
title:  "如何提升代码的可读性（+）--以python为例"
categories: coding
tags: coding
author: ble55ing
---

* content
{:toc}
## 如何提升代码的可读性（+）--以python为例

最近总是遇到不得不编程的情况，为了能够写出不会让几个月后的自己看不懂的代码，决定开始写这篇文章，来记录一下编程心得。

### 将语句简单化：lambda的使用

使用lambda语句，将多行语句精简为1行，提升代码可读性。

lambda是一个精简的匿名函数，其后跟的是参数，冒号后面跟返回值。

```
#下列语句可以将CODE字典进行翻转
UNCODE = dict(map(lambda t:(t[1],t[0]),CODE.items()))
```
### 全局变量不要设置过多

全局变量一般是影响整个程序的关键变量，如果一个变量从头用到尾，可以尝试设置为全局，否则最好设置为局部变量，如果实在不方便的，也可通过文件存储的方式进行变量传输。

因为全局变量使用时还要声明global，如果出错的话后期需要修改，如果是全局变量修改起来的工程量相对更大，会使得代码维护困难。

### 库函数/迭代器的使用

#### Counter 计数器

使用方法如下所示

```python
import collections
counter = collections.Counter(['11','22','33','11'])
#也可以直接声明为Counter(11=2, 22=1,33=1)
print(counter)
# Counter({'11': 1, '22': 1, '33': 1})
#获取其中数据的方法
print(list(counter.elements()))#根据元素计数重排的list['11', '11', '22', '33']，elememts返回的是一个迭代器
print(counter.items())#item返回一个dict_items类型dict_items([('11', 2), ('22', 1), ('33', 1)])
for k,v in counter.items():#获取dict_items中的数据
    print(k,v)
```

注意：counter中的数并不一定是正数和0，也可以通过减操作变成负数，但两个counter相减只会保存正数计数的元素。

#### yield 迭代器

 类似于return，可以用来当函数的返回值使用，其与return的区别在下面说明：

```
def func():
    print("func start")
    i=0
    while True:
        yield 0
        i+=1
        print("res:",i)
res = func()
print("A =====")
print(next(res))
print("*****")
print(next(res))
print(next(res))

#其运行结果为
A =====
func start
0
*****
res: 1
0
res: 2
0
```

当res = func()时，函数并没有真正执行，而是首先返回一个生成器，而在此时函数func并没有执行，在第一次执行next函数的时候，这个函数才真正执行，然后执行到yield，就相当于return了。而函数就停留在yield执行之后的位置，next函数第二次执行时就继续yield之后的语句。

#### glob目录文件遍历

使用glob可以匹配目录下的所有文件名

```
import glob
for name in glob.glob('dir/*'):
    print (name)
```

