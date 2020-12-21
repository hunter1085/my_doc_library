# sed 中文手册
---
## 目录
1. [简介](#introduction)
2. [运行sed](#run_sed)  
2.1 [概述](#run_overview)  
2.2 [命令行(CLI)选项](#cli_option)  
2.3 [退出码](#exit_code)  
3. [sed 脚本]()  
3.1 [sed脚本概述]()  
3.2 [sed命令总结]()  
3.3 [命令 s]()  
3.4 [常用命令]()  
3.5 [不常用命令]()  
3.6 [sed 专家的常用命令伴侣]()  
3.7 [GNU特有命令]()  
3.8 [复合命令语法]()  
3.8.1 [新起一行的命令]()  
4. [寻址: 行选择]()  
4.1 [寻址概述]()  
4.2 [基于数字的行选择]()  
4.3 [基于文本匹配的行选择]()  
4.4 [范围寻址]()  
5. [正则表达式: 文本选择]()  
5.1 [sed正则表达式概述]()  
5.2 [基础正则表达式(BRE)和扩展正则表达式(ERE)]()  
5.3 [基础正则表达式语法概述]()  
5.4 [扩展正则表达式语法概述]()  
5.5 [字符组和方括号表达式]()  
5.6 [正则表达式扩展]()  
5.7 [反向引用和子表达式]()  
5.8 [转义]()  
5.8.1 [转义优先]()  
5.9 [多字节字符及本地化语言的考虑]()  
5.9.1 [非法多字节字符]()  
5.9.2 [大小写转换]()  
5.9.3 [多字节正则表达式字符组]()  
6. [sed 高级篇: 循环和缓存]()  
6.1 [sed是怎么工作的？]()  
6.2 [???]()  
6.3 [用D,G,H,N,P处理多行的技巧]()  
6.4 [分支和流控]()  
6.4.1 [分支和循环]()  
6.4.1 [分支实例: 多行合并]()  
7. [sed 实例]()  
7.1 [多行合并]()  
7.2 [行居中]()  
7.3 [给数字加1]()  
7.4 [用小写字符重命名文件]()  
7.5 [打印Bash环境变量]()  
7.6 [逆序打印一行字符]()  
7.7 [多行文本中搜索指定文本]()  
7.8 [调整一行文本长度]()  
7.9 [逆序输出文件的每一行]()  
7.10 [增加行号]()  
7.11 [增加非空行行号]()  
7.12 [字符统计]()  
7.13 [单词统计]()  
7.14 [行数统计]()  
7.15 [打印第一行]()  
7.16 [打印最后一行]()  
7.17 [删除重复行]()  
7.18 [输出重复输入的行]()  
7.19 [删除所有重复行]()  
7.20 [删除空行]()  
8. [GNU sed的一些限制和非限制]()
9. [其他学习sed的资料]()
10. [报告缺陷]()
#### [附录A GNU 自由文档许可]()  
#### [sed概念索引]()  
#### [sed命令和选项索引]()  

<span id="introduction"></span>
# 1. 简介
 
**sed** 是一种流编辑器。所谓流编辑器，是指它能对输入的流(文件或管道输入)进行基本的文本变换。它有点像**ed**——你可以提供脚本控制**sed**处理文本的方式，但是它更高效，因为sed每次只处理一行输入。
**sed**可以接受管道输入，可以和很多命令行工具无缝连接，这个能力使他从众多的编辑器中脱颖而出。

<span id="run_sed">运行sed</span>
# 2. 运行sed

本章内容重点讲述如何使用**sed**。 **sed**脚本及**sed**内部命令请参考后续章节。本章内容包括：  
+ [概述](#run_overview)  
+ [命令行(CLI)选项](#cli_option)  
+ [退出状态](#exit_status)  

<span id="run_overview"></span>
## 2.1 概述

通常，我们用下面这种方式启动 **sed** :  
`sed SCRIPT INPUTFILE...`  
比如，下面的命令将把input.txt文件里所有的"hello"替换为"world"：  
`sed 's/hello/world/' input.txt > output.txt`  
如果你不指定输入文件，或者指定输入为 **-** ，那么 **sed** 将标准输入作为输入，下面三条命令等价:  
```
sed 's/hello/world/' input.txt > output.txt  
sed 's/hello/world/' < input.txt > output.txt  
cat input.txt | sed 's/hello/world/' - > output.txt  
```
通常，**sed** 将输出打印到标准输出，但你也可以指定"-i"，告诉 **sed** 对输入文件"原地"修改。你也可以使用W以及s///w命令，将输出写到其他指定文件，具体请参考后续章节。下面这条命令将修改file.txt，
但不产生任何输出:  
`sed -i 's/hello/world/' file.txt`  
默认情况下，**sed** 会打印出所有已经处理过(注意：这样使用"处理"而不是"修改"，实际上 **sed** 一次处理一行，默认情况下，所有的输入都会被"处理")的输入，除非输入被 **sed** 显式的删除了(比如使用d命令)，
你可以使用"-n"选项来抑制 **sed** 打印输出，同时结合p命令输出指定行，比如，下面的命令只会输出file.txt的第45行:  
`sed -n '45p' file.txt`  
**sed** 可以同时接受多个输入文件，默认情况下，**sed** 视这些文件为一个整体，下面的例子可以很好的说明这一点，这个例子将输出第一个文件(one.txt)的第1行和最后一个文件(three.txt)的最后一行。
如果希望每个文件单独处理，可以使用"-s"选项:  
`sed -n  '1p ; $p' one.txt two.txt three.txt`  
**sed** 通过"-e"和"-f"指定脚本内容，命令行中的所有其他非选项字符串被当做输入文件，如果既没有"-e"也没有"-f"，那么 **sed** 会把第一个非选项字符串当作脚本，而后续所有非选项字符串当作输入文件；
"-e"和"-f"不仅可以混合使用，而且可以多次使用(最终脚本由所有单独脚本组合而成)，请看下面的例子，它们是等价的:
```
sed 's/hello/world/' input.txt > output.txt

sed -e 's/hello/world/' input.txt > output.txt
sed --expression='s/hello/world/' input.txt > output.txt

echo 's/hello/world/' > myscript.sed
sed -f myscript.sed input.txt > output.txt
sed --file=myscript.sed input.txt > output.txt
```

<span id="cli_option"></span>
## 2.2 命令行(CLI)参数

运行 **sed** 的完整格式如下：  
`sed OPTIONS... [SCRIPT] [INPUTFILE...]`  
**sed** 可以使用如下命令行参数：  
* --version:   
  打印 **sed** 版本和版权信息，然后退出  
* --help：  
  打印 **sed** 用法包括CLI选项和bug report地址等，然后退出  
* -n  
* --quiet  
* --silent  
  通常，**sed** 会在每个脚本处理周期结束后，自动输出模式空间([6.1sed是怎么工作的]())中的内容。这些选项用于阻止这种行为，配合"p"命令可以选择性的输出指定行。  
* --debug  
  以规范的格式输出sed内部运行的程序(即调试信息)。比如：  
  ```
          $ echo 1 | sed '\%1%s21232'
          3

          $ echo 1 | sed --debug '\%1%s21232'
          SED PROGRAM:
            /1/ s/1/3/
          INPUT:   'STDIN' line 1
          PATTERN: 1
          COMMAND: /1/ s/1/3/
          PATTERN: 3
          END-OF-CYCLE:
          3
  ```
* -e SCRIPT  
* --expression=SCRIPT  
  指定要执行的命令  
* -f SCRIPT-FILE  
* --file=SCRIPT-FILE  
  指定要执行的脚本文件  
* -i[SUFFIX]  
* --in-place[=SUFFIX]  
  该选项用于指示 **sed** 直接修改原文件，即所谓的"原地(in-place)修改"模式。注意，一旦使用了该选项，"-s"选项也就自动启用了。
  sed
  
  
  
  
  
  
  
  
<span id="exit_status"></span>
## 2.3 退出码
