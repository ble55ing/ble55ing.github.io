---
layout: post
title:  "clang好用的一些命令汇总"
categories: linux
tags:  clang CodeAnalyze
author: ble55ing
---

* content
{:toc}

## clang源码分析器

clang提供了源码分析器的命令，可以获得一些分析结果通过clang -cc1 -analyze进行分析

-analyzer-checker-help查看分析器，其中可通过analyzer-checker=xxx的方式进行使用，
如=

debug.ViewCFG能够获得程序的控制流图（基本块的），
debug.ViewCallGraph能够得到函数调用图（没有实体的函数不在其中）,
debug.Stats显示基本块的可达情况
debug.DumpCalls，显示函数调用情况，不区分函数的，如果函数间存在调用关系才会以缩进的形式体现
上述的这些View都可以换成dump来获得文字版，View会生成dot结尾的图，可以通过Graphviz 来生成可视化的图，其命令为dot xx.dot -Tpng -o xx.png 。但ubantu下生成的图导到window电脑上就打不开了不知道为什么

而AST的生成可以不通过分析器而直接使用，如-ast-view可以生成ast的图，

Clang对于函数调用的处理不够完整，对于特殊情况的函数调用处理不好，如

extern 函数调用，无法获知函数体，不会创建到CallGraph中

函数指针调用，无法获知函数体，不会创建到CallGraph中

模板函数调用，在编译期可以获知进行模板实例化，会添加到CallGaph中

且调用图上没有在别的c文件中定义的函数（.h中声明的），即没有正文的函数

## 分析效果

CFG控制流图

![](https://github.com/ble55ing/PicGo/blob/master/CFG.png)

CallGraph函数调用图

![](https://github.com/ble55ing/PicGo/blob/master/CallGraph.png)


AST图

![](https://github.com/ble55ing/PicGo/blob/master/ast.png)

Stats

```
warning: main -> Total CFGBlocks: 9 | Unreachable CFGBlocks: 0 | Exhausted Block: no | Empty WorkList: yes
```

DumpCalls

```
scanf("%s", buf)
Returning conj_$3{int}
func1(3)
Returning void
func2((char *)buf)
 strcpy(s, str)
 Returning &SymRegion{conj_$9{char *}}
 func(5)
 Returning void
Returning void
func1(3)
Returning void
func(2)
Returning void
aa(1)
Returning conj_$13{int}
func(1)
Returning void
func(0)
Returning void
aab(8)
Returning conj_$17{int}
printf("a==4")
Returning conj_$3{int}
printf("a==3")
Returning conj_$6{int}
printf("a==2")
Returning conj_$9{int}
printf("a==1")
Returning conj_$12{int}
ab(a)
 printf("in ab")
 Returning conj_$15{int}
Returning 0 S32b

```

## 分析源码

所分析的源码

```
#include <stdio.h>
#include <string.h>
#include "11.h"
int global=1;
void func (int i){
    if (i==0) global=0;
    else if (i==1){
        global +=2;
        global *=2;
    }
    else{
    	global*=3;
    }
}
void func2(char* str){
	char s[20];
	strcpy(s,str);
	global +=10000;
	func(5);
}
void func1(int i){
    
	while(i<5){
		i++;
		global/=2;
	}
}
int main()
{
	
	UINT8 buf[0x200];
	int mark = 20;
    scanf("%s",buf);
    switch(buf[0]){
    case '0':
        func(0);
	aab(8);
        break;
    case '1':
	aa(1);
        func(1);
        break;
    case '2':
        func(2);
        break;
    case '3':
        func2((char*)buf);
    default:
        func1(3);
    }
	return 0;

}


```

所有生成的相关文件会放到github的clang中，包括clang的分析命令和分析器的一些选项。
[https://github.com/ble55ing/clang]https://github.com/ble55ing/clang
## 遇到的一些问题

在低版本的clag中，可以直接clang -cc1 -analyze -cfg-dump 1.c来获得程序控制流图，但较高版本后将其中一部分提出来放到了分析器中 

另外clang -cc1默认仅限当前目录，所以会出现fata error: 'stdio.h' file not found 的情况。

解决方法是使用-I添加包含库，

 1 clang -cc1 -I/usr/include -I/home/blessing/clang-llvm/build386/lib/clang/5.0.0/include -analyze -analyzer-checker=debug.DumpCFG 1.c 

第一个包含库中含有stdio.h,第二个库中有stdder.h，之后还有需要的库还可以继续添加。

如果想要将结果输出到文件中，可以在终端先输入 script -f CFG.txt ，这样就可以将当前的命令行的输出全写到文件中去了
