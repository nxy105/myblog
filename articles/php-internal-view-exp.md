# PHP 内核分析经验谈：工具篇

最近，我在分析 PHP 内核的过程中，使用到一些工具，总结了这些工具的使用方法，分享给大家。

### VLD

VLD（Vulcan Logic Dumper）是 PHP 的一个扩展。它以钩子的方式嵌入到 Zend 引擎中，收集并打印 PHP 脚本编译时期产生所有的 OPCODE。使用它，我们可以很方便地查看 PHP 源码产生的 OPCODE。

#### 安装 VLD 扩展

通过 Github 直接安装 VLD 扩展。

```
git clone https://github.com/derickr/vld.git
cd vld
phpize
./configure
make && make install
```

#### 使用 VLD 查看 OPCODE

通过一个简单的样例我们来下如何使用 VLD。首先，我们进入 `~/test/php` 目录，创建一个简单的 PHP 脚本，保存为 `simple.php`。

```
<?php

$a = 1;
$b = $a + 1;

echo $b;
```

在命令行里输入以下命令：

```
php -dvld.active=1 ~/test/php/simple.php
```

可以看到 VLD 扩展将产生的 OPCODE 打印到了终端上：

```
Finding entry points
Branch analysis from position: 0
Jump found. Position 1 = -2
filename:       /Users/joshua/test/php/simple.php
function name:  (null)
number of ops:  5
compiled vars:  !0 = $a, !1 = $b
line     #* E I O op                           fetch          ext  return  operands
-------------------------------------------------------------------------------------
   3     0  E >   ASSIGN                                                   !0, 1
   4     1        ADD                                              ~3      !0, 1
         2        ASSIGN                                                   !1, ~3
   6     3        ECHO                                                     !1
         4      > RETURN                                                   1

branch: #  0; line:     3-    6; sop:     0; eop:     4; out1:  -2
path #1: 0,
```

我们可以看到这些信息：

- `compiled vars` 表示编译时期生成所有变量。
- `filename` 当前文件名称。
- `function name` 当前所在的方法名称。
- `number of ops` 当前方法所有 OPCODE 总数。
- `opcode line` 区域是当前方法内所有 OPCODE 列表。我们依次来看看每列的含义：`line` 表示对应源码的行号；`op` 表示对应 OPCODE；`return` 表示返回的变量；`operands` 是操作数列表（一般的 OPCODE 包含1 ~ 2个操作数）。

`-dvld.active=1` 是 VLD 的基础参数，表示激活 VLD 模式。VLD 还支持其他参数，参数详情可以参考 [VLD扩展使用指南](http://www.phppan.com/2011/05/vld-extension/) 这篇文章，我在此引用其部分内容，方便查阅：

> - `-dvld.active` 是否在执行 PHP 时激活 VLD 挂钩，默认为0，表示禁用。可以使用 `-dvld.active=1` 启用。
> - `-dvld.skip_prepend` 是否跳过 php.ini 配置文件中 `auto_prepend_file` 指定的文件，默认为0，即不跳过包含的文件，显示这些包含的文件中的代码所生成的中间代码。此参数生效有一个前提条件：`-dvld.execute=0`。
> - `-dvld.skip_append` 是否跳过 php.ini 配置文件中 `auto_append_file` 指定的文件，默认为0，即不跳过包含的文件，显示这些包含的文件中的代码所生成的中间代码。此参数生效有一个前提条件：`-dvld.execute=0`。
> - `-dvld.execute` 是否执行这段 PHP 脚本，默认值为1，表示执行。可以使用 `-dvld.execute=0`，表示只显示中间代码，不执行生成的中间代码。
> - `-dvld.format` 是否以自定义的格式显示，默认为0，表示否。可以使用 `-dvld.format=1`，表示以自己定义的格式显示。这里自定义的格式输出是以 `-dvld.col_sep`指定的参数间隔。
> - `-dvld.col_sep` 在 `-dvld.format`参数启用时此函数才会有效，默认为 “\t”。
> - `-dvld.verbosity` 是否显示更详细的信息，默认为1，其值可以为0 ~ 3其实比0小的也可以，只是效果和0一样，比如0.1之类，但是负数除外，负数和效果和3的效果一样 比3大的值也是可以的，只是效果和3一样。
> - `-dvld.save_dir` 指定文件输出的路径，默认路径为 /tmp。
> - `-dvld.save_paths` 控制是否输出文件，默认为0，表示不输出文件。
> - `-dvld.dump_paths` 控制输出的内容，现在只有0和1两种情况，默认为1,输出内容。

### GDB

强大的 GDB 调试工具相信不需要我过多介绍吧。接下来，主要说一说如何使用 GDB 调试 PHP 源码。

#### 开启 PHP 的调试模式

建议重新安装一个纯净的 PHP。通常我会选择直接下载 Github 上的源码。

```
~> git clone http://git.php.net/repository/php-src.git
~> cd php-src
```

由于是从 Git 仓库中直接下载的文件，需要执行 `buildconf` 命令，该命令会生成编译所需的 configure 文件。

```
~/php-src> ./buildconf
```

然后就是标准的安装的流程，这里我们打开了调试模式，并且将其他的所有扩展都禁止安装。

```
~/php-src> ./configure --disable-all --enable-debug --prefix=/Users/joshua/Tools/bin/php-dev
~/php-src> make && make install
```

可以看到，`prefix` 参数我们设置为 `/Users/joshua/Tools/bin/php-dev`，这样，这个纯净的 PHP 就被安装到这个目录下。接下来，我们介绍的内容，都基于这个目录下编译的 PHP。

#### 使用 GBD 调试代码

我们还是使用上一节创建的 `simple.php` 文件，我们进入 GDB 调试模式：

```
~/php-src> cd ~/test/php
~/php> sudo gdb --args /Users/joshua/Tools/bin/php-dev/bin/php ./simple.php
GNU gdb (GDB) 7.12
Copyright (C) 2016 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.  Type "show copying"
and "show warranty" for details.
This GDB was configured as "x86_64-apple-darwin15.6.0".
Type "show configuration" for configuration details.
For bug reporting instructions, please see:
<http://www.gnu.org/software/gdb/bugs/>.
Find the GDB manual and other documentation resources online at:
<http://www.gnu.org/software/gdb/documentation/>.
For help, type "help".
Type "apropos word" to search for commands related to "word"...
Reading symbols from /Users/joshua/Tools/bin/php-dev/bin/php...done.
```

这里有一点需要注意，我使用的环境是 MacOS，需要使用管理员权限才能成功运行 GDB。接下来将演示如何调试 PHP。

首先，我们先执行 run 命令，运行我们需要调试的程序，由于我们没有设置断点，程序会直接运行完成后退出。

```
(gdb) run
Starting program: /Users/joshua/Tools/bin/php-dev/bin/php ./simple.php
2[Inferior 1 (process 6924) exited normally]
```

然后，我们在 PHP 的入口函数中设置断点。

```
(gdb) break main
Breakpoint 1 at 0x1003d9aa0: file sapi/cli/php_cli.c, line 1199.
```

当我们再次运行程序时，我们会发现，程序在我们设置断点的地方中断了。

```
(gdb) run
Starting program: /Users/joshua/Tools/bin/php-dev/bin/php ./test.php

Breakpoint 1, main (argc=2, argv=0x7fff5fbffb38) at sapi/cli/php_cli.c:1199
1199		int exit_status = SUCCESS;
```

这时，我们可以通过 next 指令，单步执行代码：

```
(gdb) next
1204		char *ini_entries = NULL;
(gdb) next
1205		int ini_entries_len = 0;
(gdb) next
1206		int ini_ignore = 0;
(gdb) next
1207		sapi_module_struct *sapi_module = &cli_sapi_module;
```

需要结束调试，可以执行 	quit 命令离开 GDB。

```
(gdb) quit
A debugging session is active.

	Inferior 1 [process 7101] will be killed.
```

#### GBD 常用命令

我总结了一些 GDB 的常用命令，以及命令的简写形式，方便大家使用时查阅：

- `r` `run` 运行指定的程序（在执行 gdb 脚本时指定的程序）
- `b` `break` ** 在某断代码上增加断点
- `c` `continue` 运行到下一个断点
- `p` 变量名/表达式 查看变量的值/执行表达式
- `n` `next` 执行下一条语句
- `s` `step` 执行下一条语句（可以进入到方法内部）
- `finish` `fin` 跳出当前方法
- `clear` 清除下一个断点
- `delete` 清除所有断点
- `info locals` 列出所有当前上下文变量

## 总结

工欲善其事必先利其器，使用合适的工具可以让学习事半功倍。当然，工具即是方法，方法终究只是辅助，没有正确的思路作为指导，再牛逼的工具也发挥不了它应该有的作用。学习的道路还很漫长，有新的思路，从而衍生出的新工具和方法，我会在本文里继续补充，也欢迎大家介绍新的思路和工具给我。

