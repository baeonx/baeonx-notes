

## 1 grep

### 1.1 grep '^hello' input.txt

```bash
grep '^hello' input.txt
```

```c
input.txt
hello world
```

```c
int
match(char *re, char *text)
{
  // 如果是 re='^hello', text=hello world
  if(re[0] == '^')
    // re+1变成 hello，text匹配
    return matchhere(re+1, text);
  do{  // must look at empty string
    if(matchhere(re, text))
      return 1;
  }while(*text++ != '\0');
  return 0;
}

// matchhere: search for re at beginning of text
int matchhere(char *re, char *text)
{
  if(re[0] == '\0')
    return 1;
  if(re[1] == '*')
    return matchstar(re[0], re+2, text);
  if(re[0] == '$' && re[1] == '\0')
    return *text == '\0';
  // hello, text
  // *text!='\0'说明text不为空
  // re[0]==*text 如果re的首部等于text的首部就递归
  if(*text!='\0' && (re[0]=='.' || re[0]==*text))
    return matchhere(re+1, text+1);
  return 0;
}

// matchstar: search for c*re at beginning of text
int matchstar(int c, char *re, char *text)
{
  do{  // a * matches zero or more instances
    if(matchhere(re, text))
      return 1;
  }while(*text!='\0' && (*text++==c || c=='.'));
  return 0;
}
```

```c
matchhere("hello", "hello world")
  re[0]='h', re[1]='e'  → 不是 *、不是 $、不是空
  *text='h' != '\0'，re[0]='h' == *text='h'  ✓
  → matchhere("ello", "ello world")

matchhere("ello", "ello world")
  'e' == 'e'  ✓
  → matchhere("llo", "llo world")

matchhere("llo", "llo world")
  'l' == 'l'  ✓
  → matchhere("lo", "lo world")

matchhere("lo", "lo world")
  'l' == 'l'  ✓
  → matchhere("o", "o world")

matchhere("o", "o world")
  'o' == 'o'  ✓
  → matchhere("", " world")

matchhere("", " world")
  re[0] == '\0'  → 返回 1  ✓
```





## 2 memmove

`memmove(dst, src, n)` 把 `src` 开始的 n 个字节，搬到 `dst`。

关键：`src` 和 `dst` 指向的内存**可能重叠**。如果重叠，按错误方向逐字节拷贝，会把**还没读的源数据覆盖掉**。

**两种方向**

| 情况                            | 方向                 | 原因             |
| :------------------------------ | :------------------- | :--------------- |
| `src > dst`（源在目标**后面**） | **从前往后**（`++`） | 不会覆盖未读数据 |
| `src < dst`（源在目标**前面**） | **从后往前**（`--`） | 不会覆盖未读数据 |

**为什么 `src > dst` 用 `++` 安全？**

假设 `dst` 在低地址，`src` 在高地址，两者有重叠：

```c
低地址 ──────────────────────────► 高地址
        dst
        [d0][d1][d2][d3]
                    src
                    [s0][s1][s2][s3]
```

从前往后拷：先写 `dst[0] = src[0]`。

- 写 `dst[0]` 时，会不会破坏还没读的 `src`？不会，因为 `src` 在更高的地址，`dst[0]` 离 `src` 还远。
- 每一步写的位置都在**已读位置之后**，不会碰到还没读的源数据。

所以 `src > dst` 时，**从前往后（`++`）是安全的**。

**为什么 `src < dst` 必须用 `--`（从后往前）？**

假设 `src` 在低地址，`dst` 在高地址，两者重叠：

```c
低地址 ──────────────────────────► 高地址
        src
        [s0][s1][s2][s3]
                    dst
                    [d0][d1][d2][d3]
```

**如果这时还用从前往后（`++`）会怎样？**

```c
dst[0] = src[0]   →  写到 dst[0]，这个位置可能正好是 src[2] 或 src[3] 所在的地方！
```

