+++
title = "使用yara检测webshell后门程序和恶意代码"
date = 2020-05-18T00:00:00+08:00
tags = ["Security", "WebShell", "Yara"]
categories = ["Security", "WebShell", "Yara"]
draft = false
author = "7ym0n"
+++

{{< figure src="/coding.jpg" >}}


## 关于 webshell {#关于-webshell}

由于 webshell 是通过 http 协议进行访问，被远程控制的服务器或者远程主机不易发现，非常隐蔽。简单的说来，webshell 就是后端程序编写的木马后门，通常是一条后端编程语言语句或者一个文件，通常是通过一句话执行命令行进行上传下载文件、查看数据库、执行任意程序命令等。


## 基于特征的检测方式 {#基于特征的检测方式}

该方式的优点是实现原理简单，容易实现，准确率高；缺点就是新型的 webshell 无法检测，需要不断的更新规则库进行完善。由于 webshell 是通过 http 协议进行访问，检测
webshell 常见有两种工作方式：

-   (WAF)通过 web 应用防火墙进行实时的特征及规则检测
-   通过特征及规则对目录下文件进行扫描检测


## 基于 AST 语法树的检测方式 {#基于-ast-语法树的检测方式}

该方式主要使用抽象语法树进行解析，遇到危险操作进行统计分析告警。优点在于动态解析，能发现具体代码如何工作, 准确性高，缺点是该方式的实现方式复杂，局限大，如果需要对不同的后端代码程序进行检测，需要实现不同的语言的语法树解析，难度呈现指数级。


## Yara 规则快速匹配工具 {#yara-规则快速匹配工具}

[Yara](https://github.com/VirusTotal/yar)是一款旨在帮助恶意软件研究人员识别和分类恶意软件样本的开源工具（由
virustotal 的软件工程师 Victor M. Alvarezk 开发），使用 Yara 可以基于文本或二进制模式创建恶意软件家族描述信息，当然也可以是其他匹配信息。

[Yara](https://github.com/VirusTotal/yara)支持多平台，可以运行在 Windows、Linux、Mac OS X，并通过命令行界面或
[yara-python](https://github.com/VirusTotal/yara-python)扩展的 Python 脚本使用。


## Yara 使用 {#yara-使用}

使用 Yara 进行规则匹配时需要两样东西：规则文件和目标文件，目标文件可以是文件、文件夹或进程。[Yara-rules](https://github.com/Yara-Rules/rules)规则，开源社区维护了一个很好的规则库。该库包含恶意软件等规则，这里我们直接使用[webshells\_index.yar](https://github.com/Yara-Rules/rules/blob/master/webshells%5Findex.yar)及 webshells 目录下的规则即可。yara
参数如下：

```text
YARA 3.11.0, the pattern matching swiss army knife.
Usage: yara [OPTION]... [NAMESPACE:]RULES_FILE... FILE | DIR | PID

Mandatory arguments to long options are mandatory for short options too.

       --atom-quality-table=FILE        path to a file with the atom quality table
  -C,  --compiled-rules                 load compiled rules
  -c,  --count                          print only number of matches
  -d,  --define=VAR=VALUE               define external variable
       --fail-on-warnings               fail on warnings
  -f,  --fast-scan                      fast matching mode
  -h,  --help                           show this help and exit
  -i,  --identifier=IDENTIFIER          print only rules named IDENTIFIER
  -l,  --max-rules=NUMBER               abort scanning after matching a NUMBER of rules
       --max-strings-per-rule=NUMBER    set maximum number of strings per rule (default=10000)
  -x,  --module-data=MODULE=FILE        pass FILE's content as extra data to MODULE
  -n,  --negate                         print only not satisfied rules (negate)
  -w,  --no-warnings                    disable warnings
  -m,  --print-meta                     print metadata
  -D,  --print-module-data              print module data
  -e,  --print-namespace                print rules' namespace
  -S,  --print-stats                    print rules' statistics
  -s,  --print-strings                  print matching strings
  -L,  --print-string-length            print length of matched strings
  -g,  --print-tags                     print tags
  -r,  --recursive                      recursively search directories
  -k,  --stack-size=SLOTS               set maximum stack size (default=16384)
  -t,  --tag=TAG                        print only rules tagged as TAG
  -p,  --threads=NUMBER                 use the specified NUMBER of threads to scan a directory
  -a,  --timeout=SECONDS                abort scanning after the given number of SECONDS
  -v,  --version                        show version information
```


### 递归遍历检测 {#递归遍历检测}

```shell
~# yara -r ./webshell_index.yar ~/
```

{{< figure src="/webshell_detection/20200225121339.png" >}}


### 统计样本数量 {#统计样本数量}

统计查看有多少个样本。例如:

```shell
find ~/webshell/ -type f | wc -l
```

{{< figure src="/webshell_detection/20200225122313.png" >}}

统计检测到多少个样本，由于不同规则检测到重复的样本，通过 uniq 去重。例如：

```shell
~# yara -r ~/rules/webshells_index.yar  ~/webshell/ | awk -F' ' '{print $2}'|sort|uniq|wc -l
902
```

{{< figure src="/webshell_detection/20200225122339.png" >}}


## webshell 样本 {#webshell-样本}

[webshell](https://github.com/7ym0n/webshell)的样本在 github 上找到的开源样本。主要用来测试 yara 规则及与商业或开源的其他 webshell 检测工具做对比。![](/webshell_detection/20200225123036.png)


## 检测结果 {#检测结果}

在 1613 个样本中，检测识别出 902 个样本。

| 样本总数 | 检测结果 | 检测工具 |
|------|------|------|
| 1613 | 902  | yara |

随机抽样检测（该结果有一些问题，实际检测出来了，但由于命令行去重处理在 linux
下不能很好识别中文与空格的文件及文件夹名导致的，仅作参考）。

| 样本总数 | 检测结果 | 检测工具 |
|------|------|------|
| 316  | 110  | yara |

> 注意：经过随机抽样检测，在不同的情况下结果有差异。实际情况还是要根据 yara 的规则库完善情况而定。