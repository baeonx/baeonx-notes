# 1 cmake 教程

## 1.1 g++ 教程

1. **-g**  生成调试信息

```bash
# -g 选项告诉 GCC 生成能被 GNU 调试器 GDB 使用的调试信息，便于调试程序
g++ -g test.cpp -o test
```



2. **-O[n]** 优化源代码

```bash
# 优化选项告诉 g++ 对源代码进行优化，例如省略未使用的变量、将常量表达式直接替换为结果值等
# 这些操作可以减少目标文件的代码量，提高最终可执行文件的运行效率

# -O0：不做任何优化（默认级别）
g++ -O0 test.cpp -o test

# -O1：基本优化（默认优化级别）
# 在大多数情况下使程序执行得更快
g++ -O1 test.cpp -o test

# -O2：更高级别的优化
# 除了完成 -O1 的优化外，还进行指令调整等额外优化工作
# 编译速度比 -O1 慢，但产生的代码执行速度更快
g++ -O2 test.cpp -o test

# -O3：最高级别优化
# 包括循环展开和其他与处理特性相关的优化
g++ -O3 test.cpp -o test

# 注意：-O 同时减小代码长度和执行时间，其效果等价于 -O1
g++ -O test.cpp -o test  # 等同于 -O1
```



3. **-l 和 -L** - 指定库文件 | 指定库文件路径

```bash
# -l参数(小写)就是用来指定程序要链接的库，-l参数紧接着就是库名
# 在/lib和/usr/lib和/usr/local/lib里的库直接用-l参数就能链接

# 链接glog库
g++ -lglog test.cpp -o test

# 如果库文件没放在上面三个目录里，需要使用-L参数(大写)指定库文件所在目录
# -L参数跟着的是库文件所在的目录名

# 链接mytest库，libmytest.so在/home/bing/mytestlibfolder目录下
g++ -L/home/bing/mytestlibfolder -lmytest test.cpp -o test
```



4. **-I** - 指定头文件搜索目录

```bash
# -I参数用来指定头文件的搜索目录
# /usr/include目录一般是不用指定的，gcc知道去那里找
# 但是如果头文件不在/usr/include里我们就要用-I参数指定了

# 比如头文件放在/myinclude目录里，那编译命令行就要加上-I/myinclude参数
# 如果不加你会得到一个"xxxx.h: No such file or directory"的错误

# -I参数可以用相对路径，比如头文件在当前目录，可以用-I.来指定
g++ -I/myinclude test.cpp -o test

# 指定多个头文件目录
g++ -I./include -I./third_party/include test.cpp -o test
```



5. **-Wall** - 打印警告信息

```bash
# 打印出gcc提供的所有常见警告信息
g++ -Wall test.cpp -o test
```



6. **-w** - 关闭警告信息

```bash
# 关闭所有警告信息（不推荐，除非有特殊需求）
g++ -w test.cpp -o test
```



7. **-std=c++11** - 设置编译标准

```bash
# 使用 c++11 标准编译 test.cpp
g++ -std=c++11 test.cpp -o test

# 其他常用标准
g++ -std=c++14 test.cpp -o test   # C++14标准
g++ -std=c++17 test.cpp -o test   # C++17标准
g++ -std=c++20 test.cpp -o test   # C++20标准
```



8. **-o** - 指定输出文件名

```bash
# 指定生成的可执行文件名
g++ test.cpp -o myprogram        # 输出为 myprogram
g++ test.cpp -o bin/test         # 输出到 bin 目录下的 test
g++ test.cpp -o test             # 输出为 test
```



9. **-c** - 只编译不链接

```bash
# 只生成目标文件(.o)，不进行链接
# 常用于大型项目的分步编译
g++ -c test.cpp -o test.o        # 生成 test.o 目标文件
```



## 1.2 g++实战

1. **链接include文件**

![image-20260213214555803](D:\Desktop\笔记\cpp\cmake教程\cmake教程.assets\image-20260213214555803.png)

```bash
g++ main.cpp src/swap.cpp -Iinclude
```





2. **加入--Wall  -std=c++11**

```bash
g++ main.cpp src/swap.cpp -Iinclude -Wall -std=c++11 -o b.out
```

![image-20260213215142473](D:\Desktop\笔记\cpp\cmake教程\cmake教程.assets\image-20260213215142473.png)





3. **生成静态库文件**

[图片笔记](./补充说明/静态库的意义.md)

```bash
# 1.进入源文件目录
# 进入src目录下
cd src


# 2.编译源文件为目标文件
# 汇编，生成 Swap.o 文件
# -c: 只编译不链接
# -I../include: 指定头文件路径（相对路径）
g++ Swap.cpp -c -I../include


# 3.创建静态库
# 生成静态库 libSwap.a
# ar: 归档工具
# r: 插入文件到归档中（替换同名文件）
# s: 创建索引（相当于 ranlib）
ar rs libSwap.a Swap.o


# 4. 返回项目根目录
# 回到上级目录
cd ..


# 5. 链接静态库生成可执行文件
# 链接，生成可执行文件: staticmain
# -Iinclude: 指定头文件路径
# -Lsrc: 指定库文件所在目录
# -lSwap: 链接 libSwap.a 库（注意：去掉 lib 前缀和 .a 后缀）
g++ main.cpp -Iinclude -Lsrc -lSwap -o staticmain
```





4. **生成动态库文件**

[静态库与动态库的区别](./补充说明/静态库与动态库的区别.md)

```bash
# 进入src目录下
cd src

#生成动态库libSwap.so
g++ Swap.cpp -I../include -fPIC -shared -o libSwap.so

# 上面命令等价于以下两条命令
# gcc Swap.cpp -I../include -c -fPIC
# gcc -shared -o libSwap.so Swap.o

#回到上级目录
cd ..
#链接，生成可执行文件:sharemain
g++ main.cpp -Iinclude -Lsrc -1Swap -o sharemain
```



## 1.3 gdb调试器

**前言：**

- **GDB(GNU Debugger)** 是一个用来调试C/C++程序的功能强大的调试器，是Linux系统开发C/C++最常用的调试器
- 程序员可以使用GDB来跟踪程序中的错误，从而减少程序员的工作量。
- Linux 开发C/C++一定要熟悉GDB
- **VSCode** 是通过调用GDB调试器来实现C/C++的调试工作的；

Windows系统中，常见的集成开发环境（IDE），如VS、VC等，它们内部已经嵌套了相应的调试器



**GDB主要功能：**

- 设置断点(断点可以是条件表达式)
- 使程序在指定的代码行上暂停执行，便于观察
- 单步执行程序，便于调试
- 查看程序中变量值的变化
- 动态改变程序的执行环境
- 分析崩溃程序产生的core文件



**常用的调试命令**

```bash
$ (gdb) help(h) # 查看命令帮助，具体命令查询在 gdb 中输入 help + 命令

$ (gdb) run(r) # 重新开始运行文件（run-text：加载文本文件，run-bin：加载二进制文件）

$ (gdb) start # 单步执行，运行程序，停在第一行执行语句

$ (gdb) list(l) # 查看原代码（list-n：从第n行开始查看代码；list+函数名：查看具体函数）

$ (gdb) set # 设置变量的值

$ (gdb) next(n) # 单步调试（逐过程，函数直接执行）

$ (gdb) step(s) # 单步调试（逐语句：跳入自定义函数内部执行）

$ (gdb) backtrace(bt) # 查看函数的调用栈和层级关系

$ (gdb) frame(f) # 切换函数的栈帧

$ (gdb) info(i) # 查看函数内部局部变量的数值

$ (gdb) finish # 结束当前函数，返回到函数调用点

$ (gdb) continue(c) # 继续运行

$ (gdb) print(p) # 打印值及地址

$ (gdb) quit(q) # 退出GDB

$ (gdb) break + num(b) # 在第num行设置断点

$ (gdb) info breakpoints # 查看当前设置的所有断点

$ (gdb) delete breakpoints num(d) # 删除第num个断点

$ (gdb) display # 追踪查看具体变量值

$ (gdb) undisplay # 取消追踪观察变量

$ (gdb) watch # 被设置观察点的变量发生修改时，打印显示

$ (gdb) i watch # 显示观察点

$ (gdb) enable breakpoints # 启用断点

$ (gdb) disable breakpoints # 禁用断点

$ (gdb) x # 查看内存x/20xw 显示20个单元、16进制、4字节每单元

$ (gdb) run argv[1] argv[2] # 调试时命令行传参

$ (gdb) set follow-fork-mode child # Makefile项目管理：选择跟踪父子进程（fork()）
```



## 1.4 gdb调试器实战

![image-20260214162240141](D:\Desktop\笔记\cpp\cmake教程\cmake教程.assets\image-20260214162240141.png)



![image-20260214162448046](D:\Desktop\笔记\cpp\cmake教程\cmake教程.assets\image-20260214162448046.png)







![image-20260214162852720](D:\Desktop\笔记\cpp\cmake教程\cmake教程.assets\image-20260214162852720.png)

```bash
# 上图 break 13 给13打断点
# info breakpoints 展示所有断点
# print i 打印i的值
# display i 每次调试都会打印i的值，类似于一直监测
```



# 2 vscode使用

## 2.1 使用g++实现编译



![image-20260214165512131](D:\Desktop\笔记\cpp\cmake教程\cmake教程.assets\image-20260214165512131.png)

```bash
# 编译main.cpp src里的Gun.cpp Solider.cpp然后头文件是在include里面，生成myexe可执行文件
g++ main.cpp src/Gun.cpp src/Solider.cpp -Iinclude -o myexe
```





![image-20260214165801385](D:\Desktop\笔记\cpp\cmake教程\cmake教程.assets\image-20260214165801385.png)

```bash
#多添加-Wall实现warning打印
g++ main.cpp src/Gun.cpp src/Solider.cpp -Iinclude -o myexe -Wall -g -o2

```



## 2.2 使用cmake实现编译

![image-20260214170228336](D:\Desktop\笔记\cpp\cmake教程\cmake教程.assets\image-20260214170228336.png)



```cmake
# 指定cmake最小版本
cmake_minimum_required(VERSION 3.0)

# 指定项目名称
project(SOLIDERFIRE)

# 指定include目录
include_directories(${CMAKE_SOURCE_DIR}/include)

# 将生成的可执行文件命名为my_cmake_exe，加入编译的cpp文件为main.cpp src/Gun.cpp src/Solider.cpp
add_executable(my_cmake_exe main.cpp src/Gun.cpp src/Solider.cpp)

# 原有的编译选项基础上追加三个新的编译选项
set (CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -g -O2 -Wall")
```





![image-20260214170344288](D:\Desktop\笔记\cpp\cmake教程\cmake教程.assets\image-20260214170344288.png)

```bash
mkdir build 
cd build/
cmake ..
# 通过进入build文件夹，cmake..编译上层目录下的cmakeLists，让cmakeLists产生的东西放到build文件夹当中
make

# 假如修改了mian.cpp,再执行make只会重新编译main.cpp，不会重新编译别的cpp，对于大型文件而言，这是其优点
make

# 清理之前的构建文件
rm -rf *
```





























