### gdb调试技巧

参考文章:

- [GDB调试入门指南](https://zhuanlan.zhihu.com/p/74897601)

调试方式运行程序:

```shell
gdb helloWorld
```

启动后设置参数:

```
$ gdb helloWorld
$ set args 1 2 3
$ run
```

**直接调试相关id进程**

```java
$ gdb hello 20829
$ gdb hello --pid 20829
```

**attach方式: 调试已运行的进程**

```shell
$ gdb
(gdb) file hello
Reading symbols from hello...done.
(gdb) attach 20829
```

**运行**

- run：简记为 r ，其作用是运行程序，当遇到断点后，程序会在断点处停止运行，等待用户输入下一步的命令。
- continue （简写c ）：继续执行，到下一个断点处（或运行结束）
- next：（简写 n），单步跟踪程序，当遇到函数调用时，也不进入此函数体；此命令同 step 的主要区别是，step 遇到用户自定义的函数，将步进到函数中去运行，而 next 则直接调用函数，不会进入到函数体内。
- step （简写s）：**单步调试如果有函数调用，则进入函数**；与命令n不同，n是不进入调用的函数的
- until：当你厌倦了在一个循环体内单步跟踪时，这个命令可以运行程序直到退出循环体。
- until+行号： **运行至某行，不仅仅用来跳出循环**
- finish： **运行程序，直到当前函数完成返回，并打印函数返回时的堆栈地址和返回值及参数值等信息**
- call 函数(参数)：**调用程序中可见的函数，并传递“参数”，如：call gdb_test(55)**
- quit：简记为 q ，退出gdb

**查看源码**

- list ：简记为 l ，其作用就是列出程序的源代码，默认每次显示10行。

- list 行号：将显示当前文件以“行号”为中心的前后10行代码，如：list 12

- list 函数名：将显示“函数名”所在函数的源代码，如：list main

- list ：不带参数，将接着上一次 list 命令的，输出下边的内容

- 设置源码一次列出的行数

  每次只显示10行，那么有没有方法每次列出更多呢？

  我们可以通过listsize属性来设置，例如设置每次列出20行:

  ```shell
  (gdb) set listsize 20
  (gdb) show listsize
  Number of source lines gdb will list by default is 20.
  ```

**设置断点**

- **break n （简写b n）:在第n行处设置断点**

  **(可以带上代码路径和代码名称： b  test.c:578)**

- b fn1 if a＞b：条件断点设置, 如` break test.c:6 if num>0`

- break func（break缩写为b）：在函数func()的入口处设置断点，如：break cb_button

- delete 断点号n：删除第n个断点

- disable 断点号n：暂停第n个断点

- enable 断点号n：开启第n个断点

- clear 行号n：清除第n行的断点

- info b （info breakpoints） ：显示当前程序的断点设置情况

- delete breakpoints：清除所有断点：

**打印表达式**

- print a：将显示整数 a 的值

- print ++a：将把 a 中的值加1,并显示出来

- print name：将显示字符串 name 的值

- print gdb_test(22)：将以整数22作为参数调用 gdb_test() 函数

- print gdb_test(a)：将以变量 a 作为参数调用 gdb_test() 函数

- **display 表达式：在单步运行时将非常有用，使用display命令设置一个表达式后，它将在每次单步进行指令后，紧接着输出被设置的表达式及值。如： display a**

- **watch 表达式：设置一个监视点，一旦被监视的“表达式”的值改变，gdb会暂停执行，并把遍历old值和new值都会显示出来。如： watch a**

- whatis ：查询变量或函数

- <font style="color:red">扩展info locals： 显示当前堆栈页的所有变量</font>

- **查看指针**

  ```shell
  (gdb) print *buf
  $3 = 110 'n'
  ```

  仅仅使用`*`只能打印第一个值,如果要打印多个字符值,可以后面跟上`@`并加上打印的长度，或者@后面跟上变量值。如:

  ```shell
  (gdb) print *buf@i
  $4 = 110 'nwlrbbmqb'
  
  (gdb) p *d
  $2 = 0
  (gdb) p *d@10
  $3 = {0, 1, 2, 3, 4, 5, 6, 7, 8, 9}
  ```

  **另外值得一提的是，`$`可表示上一个变量，在调试链表时时经常会用到的，它有next成员代表下一个节点，则可使用下面方式不断打印链表内容**

  ```shell
  p *linkNode #这里显示linkNode节点内容
  p *$.next #这里显示linkNode节点下一个节点内容
  ```

更多用法参考文章: [GDB调试入门指南](https://zhuanlan.zhihu.com/p/74897601)

### gdb 调试coredump

#### 设置core文件大小

系统默认不生成core文件，需要进一步在ulimi中设置core文件大小。

```shell
$ ulimit -a
core file size          (blocks, -c) 0 # 默认不生产core file
data seg size           (kbytes, -d) unlimited
scheduling priority             (-e) 0
...
```

使用命令**`ulimit -c unlimited`**设置core file size无限制

```shell
$ ulimit -c unlimited
$ ulimit -a
core file size          (blocks, -c) unlimited
data seg size           (kbytes, -d) unlimited
scheduling priority             (-e) 0
...
```

永久设置:

```shell
$ vim /etc/security/limits.conf
* soft core unlimited
* soft hard unlimited
```

#### 设置core文件保存位置

查看core文件保存位置:

```shell
$ cat /proc/sys/kernel/core_pattern
/data/corefile/core_%e_%t
```

临时设置core文件保存位置:

```shell
$ echo "/data/corefile/core_%e_%t" > /proc/sys/kernel/core_pattern
```

永久设置core文件保存位置:

```shell
$ vim /etc/sysctl.conf
kernel.core_uses_pid = 0
kernel.core_pattern= /data/corefile/core_%e_%t

$ sysctl -p
```

core dump命名方式中pattern的含义:

- `%p` 进程ID
- `%u`实际用户ID
- `%g`进程的租ID
- `%s`导致本次core dump的信号
- `%t` core dump的时间(1970年1月1日计起的秒数)
- `%h` 主机名
- `%e` 程序文件名

### gdb调试coredump尝试

语法:**gdb 可执行文件 core文件**，如:`gdb ./demo01 /data/corefile/core_demo01_1673508115`

- `bt`、`where`查看栈信息

  ```shell
  gdb$ bt
  #0  0x00007fdb87b774dc in free () from /lib64/libc.so.6
  #1  0x000000000050f558 in dumpTest::test (this=0x7ffe9d008a50) at /data/home/lukexwang/code/c++Code/demo01/generalTest.cpp:35
  #2  0x00000000004d0d2d in main (argc=0x1, argv=0x7ffe9d0090a8) at /data/home/lukexwang/code/c++Code/demo01/demo01.cpp:417
  gdb$ where
  #0  0x00007fdb87b774dc in free () from /lib64/libc.so.6
  #1  0x000000000050f558 in dumpTest::test (this=0x7ffe9d008a50) at /data/home/lukexwang/code/c++Code/demo01/generalTest.cpp:35
  #2  0x00000000004d0d2d in main (argc=0x1, argv=0x7ffe9d0090a8) at /data/home/lukexwang/code/c++Code/demo01/demo01.cpp:417
  ```

- `f 1`切换栈帧,因为上面的`f 0`是系统`libc.so.6`的代码，不是我们的代码。所以我们需要切换为`f 1`

  