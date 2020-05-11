---
layout: post
title:  "clang插桩及相关静态分析"
categories: CodeAnalyze
tags:  clang
author: ble55ing
---

* content
{:toc}

## clang源码插桩

前文提到了clang的源码插桩，本文来对源码插桩做一些辅助的分析。

## 函数调用关系图

在使用clang生成函数调用图时曾分析过，clang的函数调用图生成无法识别不同源文件的函数，这在对软件进行插桩的时候就会遇到很大的问题，所以需要自己实现这部分功能。具体思路为在找到函数体的定义时，记录下函数名称，并在之后Stmt块遍历时，如果是CallExpr，即函数调用关系，则记录下调用的函数，形成函数的一对调用关系。当遍历完全部的函数之后，即生成了全部的函数调用关系。

## 函数与插桩的关系

为了之后插桩后能够用的较为方便，需要获取一些函数与插桩点间的关系

首先需要函数与插桩点的对应关系，即获悉哪些插桩点是属于哪些函数的，方便进行粗粒度（函数层面）和细粒度（插桩点层面）的分析。具体思路为在找到函数体定义时记录下当前的插桩点下标。

然后还要获得函数调用的信息，即在程序运行起来后是否成功执行到了函数调用，具体思路为在Stmt块插桩的基础上，添加在函数开始位置的插桩点，并记录产生函数调用时的插桩点下标，标记基本块与函数调用图之间的关系。

## 基本块间的可达关系

插桩点并不是都能够在一次运行中被覆盖到的，比较明显的就是有些插桩点之间有并列的关系，如if-else结构和switch-case结构，只能选择其中的一条路径，而其他的路径就不能达到。因此需要将这些并列的情况分析出来，方便进行基本块间的结构分析。

首先是对于if-else结构的处理：如果If->getElse()是存在的，那证明这是if-else并还没有结束，需要记录并继续访问；如果是else结构或else if的最后一个，则需要标识为结束位置。

```
Stmt *EL = If->getElse();
		if (EL && !isa<IfStmt>(EL)) //else
		{
			ChooseOne(EL,-1);//end
			InstrumentStmt(EL, flag);
		}
		else if (EL){
			ChooseOne(EL,0);
			IfStmt *IfEL = cast<IfStmt>(EL);
			if(!IfEL->getElse()) ChooseOne(EL,-1);//end
		}
```

然后是对于switch-case结构的处理。原先在找到switch-case结构时使用的是while循环来处理，即找到SwitchStmt的时候就直接对其中的CaseStmt和DefaultStmt进行处理。好处在于这样做在代码上看着比较简洁，但问题在于整体处理结构上与别的模块差别较大，一开始只有插桩时还好，后面添加源码分析就需要每个功能都再针对写一遍，很麻烦，所以将Switch-case结构分为了对SwitchStmt、CaseStmt和DefaultStmt的处理。

