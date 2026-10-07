# xv6 pingpong 调试完整流程

## 第 0 步：准备（确认环境）

```bash
# SSH 进容器，先确认编译通过
cd /home/xv6
make                 # 确保 user/_pingpong 已生成
```

然后开两个 SSH 会话：

```bash
ssh ubuntu@localhost -p 2233   # 连两次
```



## 第 1 步：窗口 1 启动 qemu-gdb

```bash
cd /home/xv6
make qemu-gdb
```

会看到：

```text
*** Now run 'gdb' in another window.
qemu-system-riscv64 ... -S -gdb tcp::26000
```

然后卡住不动了——这是正常的，qemu 暂停在第一个指令等你连 gdb。此时窗口 1 没有 `$` 提示符。

## 第 2 步：窗口 2 启动 gdb 并连接

```bash
cd /home/xv6
gdb-multiarch
```

在 gdb 里逐条输入：

```bash
set architecture riscv:rv64
target remote 127.0.0.1:26000
```

会看到：

```text
Remote debugging using 127.0.0.1:26000
0x0000000000001000 in ?? ()
```

说明连上了。再加载 pingpong 符号：

```bash
file user/_pingpong
```



## 第 3 步：先让系统完全启动（关键，避开 init/sh 噪音）

```bash
c
```

立刻切到窗口 1，应该看到 xv6 启动信息，最后出现：

```bash
init: starting sh
$
```

等 `$` 出现后，切回窗口 2，按 `Ctrl+C` 把系统暂停（会显示 `Thread ... received signal SIGINT`）。

为什么先启动完再设断点：所有用户程序的代码都在地址 0 附近，开机时 init/sh 也会经过。先把系统跑到 shell，再设断点，就不会被它们误触发。

## 第 4 步：设断点（现在设，只针对 pingpong）

```bash
b pingpong.c:21     # 断在第 21 行 pipe(f)
c
```

切回窗口 1，在 `$` 提示符输入：

```bash
pingpong
```

回车后，切回窗口 2，gdb 会在第 21 行停住：

```bash
Breakpoint 2, main (...) at user/pingpong.c:21
21    pipe(f);
```



## 第 5 步：开始单步调试

命中后，用这些命令：

```
n                  # 执行当前行(不进入函数),再按 n 继续下一行
s                  # 单步进入函数(比如想看 fork/pipe 内部)
p f[0]             # 打印变量,比如 pipe 后的读端 fd
p f[1]             # 写端 fd
p pid              # fork 返回值,子进程=0
bt                 # 调用栈
info registers     # 看寄存器
x/10i $pc          # 看当前指令
```

比如想走父进程路径：

```
b pingpong.c:31    # 断父进程的 write(f[1], ...)
c
```



想验证"字节传没传过去"：

```
b pingpong.c:25    # 断子进程的 read
c

# 停住后:
p zread
```



## 第 6 步：改代码后重调试

改完 pingpong.c 想重新调试：

```
# 窗口2(gdb)里:
kill                 # 结束当前调试

# 窗口1 重新 make qemu-gdb(会重新编译)

# 窗口2: target remote 127.0.0.1:26000
#        file user/_pingpong  (重新加载,因为二进制变了)
```



## 常见坑

| 情况                              | 可能原因 / 处理                              |
| :-------------------------------- | :------------------------------------------- |
| `c` 后窗口1 没反应                | 检查 qemu 是否已启动完成、断点是否命中       |
| 窗口1 输入 `pingpong` 后 gdb 没停 | 断点没设对行号，或文件路径不匹配             |
| 命中别的进程                      | 断点地址被 init/sh 共用，先跑到 `$` 再设断点 |
| 单步 `n` 慢/卡                    | 用户态单步经过内核返回路径，正常现象         |

你先按这个流程走一遍，把每一步的输出（尤其是第 4 步命中后 `p f[0]` / `p f[1]` / `p pid` 的值）贴给我，我帮你定位 pingpong 的问题。