---
layout: post
title:  "clang插桩"
categories: linux
tags:  clang CodeAnalyze
author: ble55ing
---

* content
{:toc}

## clang+llvm框架

clang是一套结构化编译器的前端，llvm是一个底层虚拟机后端，整体组成了一个编译器架构。clang进行词法分析和语法分析，然后交由llvm进行目标代码的生成。

clang的静态分析很大程度上通过AST（Abstract Syntax Tree  ，抽象语法树 ）来展现，clang的模块化结构清晰，代码相对简单，所以很适合进行功能的扩展，因此选用了clang来进行源码分析和插桩。



## 抽象语法树AST

clang的抽象语法树可以很好的表示程序的结构和逻辑，是clang对源程序进行分析后的产物。接下来结合一个实例展示一下这个抽象语法树是一个什么样的概念。相关文件存放在[https://github.com/ble55ing/clang/tree/master/clang-insert](https://github.com/ble55ing/clang/tree/master/clang-insert)

首先来看一下源码，这是一个包含两个函数的源文件

````
#include "11.h"
#include <stdio.h>

int ab(int a){
    int b =0;
    if (a==b) printf("in ab");
    return 0;
}
int aa(int a){
    if (a==0) {

	for (;a<1;a++) printf("a==0");
	ab(a);
    }
    else if (a==1) printf("a==1");
    else if (a==2) printf("a==2");
    else if (a==3) printf("a==3");
    else if (a==4) printf("a==4");

    return 5;
}
````

然后我们来看一下它生成的结构化语法树

```
|-FunctionDecl 0xcc95d58 prev 0xcc3ddb0 <11.c:4:1, line:8:1> line:4:5 used ab 'int (int)'

| |-ParmVarDecl 0xcc95cc8 <col:8, col:12> col:12 used a 'int'

| `-CompoundStmt 0xcc97138 <col:14, line:8:1>

|   |-DeclStmt 0xcc95e90 <line:5:5, col:13>

|   | `-VarDecl 0xcc95e10 <col:5, col:12> col:9 used b 'int' cinit

|   |   `-IntegerLiteral 0xcc95e70 <col:12> 'int' 0

|   |-IfStmt 0xcc96090 <line:6:5, col:29>

|   | |-<<<NULL>>>

|   | |-<<<NULL>>>

|   | |-BinaryOperator 0xcc95f28 <col:9, col:12> 'int' '=='

|   | | |-ImplicitCastExpr 0xcc95ef8 <col:9> 'int' <LValueToRValue>

|   | | | `-DeclRefExpr 0xcc95ea8 <col:9> 'int' lvalue ParmVar 0xcc95cc8 'a' 'int'

|   | | `-ImplicitCastExpr 0xcc95f10 <col:12> 'int' <LValueToRValue>

|   | |   `-DeclRefExpr 0xcc95ed0 <col:12> 'int' lvalue Var 0xcc95e10 'b' 'int'

|   | |-CallExpr 0xcc96030 <col:15, col:29> 'int'

|   | | |-ImplicitCastExpr 0xcc96018 <col:15> 'int (*)(const char *, ...)' <FunctionToPointerDecay>

|   | | | `-DeclRefExpr 0xcc95f50 <col:15> 'int (const char *, ...)' Function 0xcc87a10 'printf' 'int (const char *, ...)'

|   | | `-ImplicitCastExpr 0xcc96078 <col:22> 'const char *' <BitCast>

|   | |   `-ImplicitCastExpr 0xcc96060 <col:22> 'char *' <ArrayToPointerDecay>

|   | |     `-StringLiteral 0xcc95fb8 <col:22> 'char [6]' lvalue "in ab"

|   | `-<<<NULL>>>

|   `-ReturnStmt 0xcc97120 <line:7:5, col:12>

|     `-IntegerLiteral 0xcc97100 <col:12> 'int' 0

`-FunctionDecl 0xcc97210 prev 0xcc3dc28 <line:9:1, line:21:1> line:9:5 aa 'int (int)'

  |-ParmVarDecl 0xcc97180 <col:8, col:12> col:12 used a 'int'

  `-CompoundStmt 0xcc97d00 <col:14, line:21:1>

    |-IfStmt 0xcc97c90 <line:10:5, line:18:33>

    | |-<<<NULL>>>

    | |-<<<NULL>>>

    | |-BinaryOperator 0xcc97310 <line:10:9, col:12> 'int' '=='

    | | |-ImplicitCastExpr 0xcc972f8 <col:9> 'int' <LValueToRValue>

    | | | `-DeclRefExpr 0xcc972b0 <col:9> 'int' lvalue ParmVar 0xcc97180 'a' 'int'

    | | `-IntegerLiteral 0xcc972d8 <col:12> 'int' 0

    | |-CompoundStmt 0xcc97628 <col:15, line:14:5>

    | | |-ForStmt 0xcc97510 <line:12:2, col:30>

    | | | |-<<<NULL>>>

    | | | |-<<<NULL>>>

    | | | |-BinaryOperator 0xcc97398 <col:8, col:10> 'int' '<'

    | | | | |-ImplicitCastExpr 0xcc97380 <col:8> 'int' <LValueToRValue>

    | | | | | `-DeclRefExpr 0xcc97338 <col:8> 'int' lvalue ParmVar 0xcc97180 'a' 'int'

    | | | | `-IntegerLiteral 0xcc97360 <col:10> 'int' 1

    | | | |-UnaryOperator 0xcc973e8 <col:12, col:13> 'int' postfix '++'

    | | | | `-DeclRefExpr 0xcc973c0 <col:12> 'int' lvalue ParmVar 0xcc97180 'a' 'int'

    | | | `-CallExpr 0xcc974b0 <col:17, col:30> 'int'

    | | |   |-ImplicitCastExpr 0xcc97498 <col:17> 'int (*)(const char *, ...)' <FunctionToPointerDecay>

    | | |   | `-DeclRefExpr 0xcc97408 <col:17> 'int (const char *, ...)' Function 0xcc87a10 'printf' 'int (const char *, ...)'

    | | |   `-ImplicitCastExpr 0xcc974f8 <col:24> 'const char *' <BitCast>

    | | |     `-ImplicitCastExpr 0xcc974e0 <col:24> 'char *' <ArrayToPointerDecay>

    | | |       `-StringLiteral 0xcc97468 <col:24> 'char [5]' lvalue "a==0"

    | | `-CallExpr 0xcc975e0 <line:13:2, col:6> 'int'

    | |   |-ImplicitCastExpr 0xcc975c8 <col:2> 'int (*)(int)' <FunctionToPointerDecay>

    | |   | `-DeclRefExpr 0xcc97548 <col:2> 'int (int)' Function 0xcc95d58 'ab' 'int (int)'

    | |   `-ImplicitCastExpr 0xcc97610 <col:5> 'int' <LValueToRValue>

    | |     `-DeclRefExpr 0xcc97570 <col:5> 'int' lvalue ParmVar 0xcc97180 'a' 'int'

    | `-IfStmt 0xcc97c58 <line:15:10, line:18:33>

    |   |-<<<NULL>>>

    |   |-<<<NULL>>>

    |   |-BinaryOperator 0xcc976b0 <line:15:14, col:17> 'int' '=='

    |   | |-ImplicitCastExpr 0xcc97698 <col:14> 'int' <LValueToRValue>

    |   | | `-DeclRefExpr 0xcc97650 <col:14> 'int' lvalue ParmVar 0xcc97180 'a' 'int'

    |   | `-IntegerLiteral 0xcc97678 <col:17> 'int' 1

    |   |-CallExpr 0xcc97748 <col:20, col:33> 'int'

    |   | |-ImplicitCastExpr 0xcc97730 <col:20> 'int (*)(const char *, ...)' <FunctionToPointerDecay>

    |   | | `-DeclRefExpr 0xcc976d8 <col:20> 'int (const char *, ...)' Function 0xcc87a10 'printf' 'int (const char *, ...)'

    |   | `-ImplicitCastExpr 0xcc97790 <col:27> 'const char *' <BitCast>

    |   |   `-ImplicitCastExpr 0xcc97778 <col:27> 'char *' <ArrayToPointerDecay>

    |   |     `-StringLiteral 0xcc97700 <col:27> 'char [5]' lvalue "a==1"

    |   `-IfStmt 0xcc97c20 <line:16:10, line:18:33>

    |     |-<<<NULL>>>

    |     |-<<<NULL>>>

    |     |-BinaryOperator 0xcc97808 <line:16:14, col:17> 'int' '=='

    |     | |-ImplicitCastExpr 0xcc977f0 <col:14> 'int' <LValueToRValue>

    |     | | `-DeclRefExpr 0xcc977a8 <col:14> 'int' lvalue ParmVar 0xcc97180 'a' 'int'

    |     | `-IntegerLiteral 0xcc977d0 <col:17> 'int' 2

    |     |-CallExpr 0xcc978a0 <col:20, col:33> 'int'

    |     | |-ImplicitCastExpr 0xcc97888 <col:20> 'int (*)(const char *, ...)' <FunctionToPointerDecay>

    |     | | `-DeclRefExpr 0xcc97830 <col:20> 'int (const char *, ...)' Function 0xcc87a10 'printf' 'int (const char *, ...)'

    |     | `-ImplicitCastExpr 0xcc978e8 <col:27> 'const char *' <BitCast>

    |     |   `-ImplicitCastExpr 0xcc978d0 <col:27> 'char *' <ArrayToPointerDecay>

    |     |     `-StringLiteral 0xcc97858 <col:27> 'char [5]' lvalue "a==2"

    |     `-IfStmt 0xcc97be8 <line:17:10, line:18:33>

    |       |-<<<NULL>>>

    |       |-<<<NULL>>>

    |       |-BinaryOperator 0xcc97960 <line:17:14, col:17> 'int' '=='

    |       | |-ImplicitCastExpr 0xcc97948 <col:14> 'int' <LValueToRValue>

    |       | | `-DeclRefExpr 0xcc97900 <col:14> 'int' lvalue ParmVar 0xcc97180 'a' 'int'

    |       | `-IntegerLiteral 0xcc97928 <col:17> 'int' 3

    |       |-CallExpr 0xcc979f8 <col:20, col:33> 'int'

    |       | |-ImplicitCastExpr 0xcc979e0 <col:20> 'int (*)(const char *, ...)' <FunctionToPointerDecay>

    |       | | `-DeclRefExpr 0xcc97988 <col:20> 'int (const char *, ...)' Function 0xcc87a10 'printf' 'int (const char *, ...)'

    |       | `-ImplicitCastExpr 0xcc97a40 <col:27> 'const char *' <BitCast>

    |       |   `-ImplicitCastExpr 0xcc97a28 <col:27> 'char *' <ArrayToPointerDecay>

    |       |     `-StringLiteral 0xcc979b0 <col:27> 'char [5]' lvalue "a==3"

    |       `-IfStmt 0xcc97bb0 <line:18:10, col:33>

    |         |-<<<NULL>>>

    |         |-<<<NULL>>>

    |         |-BinaryOperator 0xcc97ab8 <col:14, col:17> 'int' '=='

    |         | |-ImplicitCastExpr 0xcc97aa0 <col:14> 'int' <LValueToRValue>

    |         | | `-DeclRefExpr 0xcc97a58 <col:14> 'int' lvalue ParmVar 0xcc97180 'a' 'int'

    |         | `-IntegerLiteral 0xcc97a80 <col:17> 'int' 4

    |         |-CallExpr 0xcc97b50 <col:20, col:33> 'int'

    |         | |-ImplicitCastExpr 0xcc97b38 <col:20> 'int (*)(const char *, ...)' <FunctionToPointerDecay>

    |         | | `-DeclRefExpr 0xcc97ae0 <col:20> 'int (const char *, ...)' Function 0xcc87a10 'printf' 'int (const char *, ...)'

    |         | `-ImplicitCastExpr 0xcc97b98 <col:27> 'const char *' <BitCast>

    |         |   `-ImplicitCastExpr 0xcc97b80 <col:27> 'char *' <ArrayToPointerDecay>

    |         |     `-StringLiteral 0xcc97b08 <col:27> 'char [5]' lvalue "a==4"

    |         `-<<<NULL>>>

    `-ReturnStmt 0xcc97ce8 <line:20:5, col:12>

      `-IntegerLiteral 0xcc97cc8 <col:12> 'int' 5
```

对于两个函数，在AST中为两个FunctionDecl结构，每个结构下面延伸出不同的Stmt块，并根据不同的嵌套关系产生缩进。

如果觉得上边这个看起来比较费劲，也可以生成可视化的AST图，如下所示：

![](https://github.com/ble55ing/PicGo/blob/master/ab.png)

![](https://github.com/ble55ing/PicGo/blob/master/aa.png)



## 基于抽象语法树的插桩

整体的插桩也是基于抽象语法树进行的，是在遍历抽象语法树的过程中对源码中的关键位置插入插桩点，相当于是重写了RecursiveASTVisitor类，修改了VisitFunctionDecl和VisitStmt的功能，加入了代码插桩的部分，然后使用Rewriter将插桩代码写入到文件中。

VisitFunctionDecl函数负责找到函数之后的处理，如果f->hasBody则说明其有着函数定义的部分，接下来就将对其进行分析，如果f->isMain()则表示函数是主函数。具体的相关函数可参考http://clang.llvm.org/doxygen/classclang_1_1FunctionDecl.html

VisitStmt函数负责遍历Stmt结构块并进行相应的处理，如IfStmt，WhileStmt，ForStmt，SwitchStmt，DeclStmt，CallExpr，ReturnStmt等等，具体的结构块分类可以看http://clang.llvm.org/doxygen/classclang_1_1Stmt.html

## 参考连接

https://www.ibm.com/developerworks/cn/opensource/os-cn-clang/

http://clang.llvm.org/
