```cpp
// SPDX-FileCopyrightText: 2018 Nate Foster
// SPDX-License-Identifier: Apache-2.0
/* -*- P4_16 -*- */
#include <core.p4>
#include <v1model.p4>

const bit<16> TYPE_IPV4 = 0x800;
const bit<16> TYPE_SRCROUTING = 0x1234;

#define MAX_HOPS 9

/*************************************************************************
*********************** H E A D E R S  ***********************************
*************************************************************************/

typedef bit<9>  egressSpec_t;
typedef bit<48> macAddr_t;
typedef bit<32> ip4Addr_t;

header ethernet_t {
    macAddr_t dstAddr;
    macAddr_t srcAddr;
    bit<16>   etherType;
}

header srcRoute_t {
    bit<1>    bos;
    bit<15>   port;
}

header ipv4_t {
    bit<4>    version;
    bit<4>    ihl;
    bit<8>    diffserv;
    bit<16>   totalLen;
    bit<16>   identification;
    bit<3>    flags;
    bit<13>   fragOffset;
    bit<8>    ttl;
    bit<8>    protocol;
    bit<16>   hdrChecksum;
    ip4Addr_t srcAddr;
    ip4Addr_t dstAddr;
}

struct metadata {
    /* empty */
}

struct headers {
    ethernet_t              ethernet;
    srcRoute_t[MAX_HOPS]    srcRoutes;
    ipv4_t                  ipv4;
}

/*************************************************************************
*********************** P A R S E R  ***********************************
*************************************************************************/

parser MyParser(packet_in packet,
                out headers hdr,
                inout metadata meta,
                inout standard_metadata_t standard_metadata) {


    state start {
        transition parse_ethernet;
    }

    state parse_ethernet {
        packet.extract(hdr.ethernet);
        /*
         * TODO: Modify the next line to select on hdr.ethernet.etherType
         * If the value is TYPE_SRCROUTING transition to parse_srcRouting
         * otherwise transition to accept.
         */
        transition select(hdr.ethernet.etherType){
            TYPE_SRCROUTING: parse_srcRouting;
            TYPE_IPV4: parse_ipv4;
            default: accept;
        }
    }

    state parse_srcRouting {
        /*
         * TODO: extract the next entry of hdr.srcRoutes
         * while hdr.srcRoutes.last.bos is 0 transition to this state
         * otherwise parse ipv4
         */
        packet.extract(hdr.srcRoutes.next);
        transition select(hdr.srcRoutes.last.bos){
            0: parse_srcRouting;
            default: parse_ipv4;
        }
    }

    state parse_ipv4 {
        packet.extract(hdr.ipv4);
        transition accept;
    }

}


/*************************************************************************
************   C H E C K S U M    V E R I F I C A T I O N   *************
*************************************************************************/

control MyVerifyChecksum(inout headers hdr, inout metadata meta) {
    apply {  }
}


/*************************************************************************
**************  I N G R E S S   P R O C E S S I N G   *******************
*************************************************************************/

control MyIngress(inout headers hdr,
                  inout metadata meta,
                  inout standard_metadata_t standard_metadata) {

    action drop() {
        mark_to_drop(standard_metadata);
    }

    action srcRoute_nhop() {
        /*
         * TODO: set standard_metadata.egress_spec
         * to the port in hdr.srcRoutes[0] and
         * pop an entry from hdr.srcRoutes
         */

        standard_metadata.egress_spec = (bit<9>)hdr.srcRoutes[0].port;
        hdr.srcRoutes.pop_front(1);
    }

    action srcRoute_finish() {
        hdr.ethernet.etherType = TYPE_IPV4;
    }

    action update_ttl(){
        hdr.ipv4.ttl = hdr.ipv4.ttl - 1;
    }

    apply {
        if (hdr.srcRoutes[0].isValid()){
            /*
             * TODO: add logic to:
             * - If final srcRoutes (top of stack has bos==1):
             *   - change etherType to IP
             * - choose next hop and remove top of srcRoutes stack
             */
            if(hdr.srcRoutes[0].bos == 1)
            {
                srcRoute_finish();
            }
            srcRoute_nhop();
            // 如果先执行srcRoute_nhop()
            // 
            if (hdr.ipv4.isValid()){
                update_ttl();
            }
        }else{
            drop();
        }
    }
}

/*************************************************************************
****************  E G R E S S   P R O C E S S I N G   *******************
*************************************************************************/

control MyEgress(inout headers hdr,
                 inout metadata meta,
                 inout standard_metadata_t standard_metadata) {
    apply {  }
}

/*************************************************************************
*************   C H E C K S U M    C O M P U T A T I O N   **************
*************************************************************************/

control MyComputeChecksum(inout headers hdr, inout metadata meta) {
    apply {  }
}

/*************************************************************************
***********************  D E P A R S E R  *******************************
*************************************************************************/

control MyDeparser(packet_out packet, in headers hdr) {
    apply {
        packet.emit(hdr.ethernet);
        packet.emit(hdr.srcRoutes);
        packet.emit(hdr.ipv4);
    }
}

/*************************************************************************
***********************  S W I T C H  *******************************
*************************************************************************/

V1Switch(
MyParser(),
MyVerifyChecksum(),
MyIngress(),
MyEgress(),
MyComputeChecksum(),
MyDeparser()
) main;

```





# 第 7 课：源路由 —— 让"路径"由源主机说了算

> 本课对应大纲第 7 课。第 6 课学了头栈（header stack）和逐跳 push 记录路径（MRI/INT）。
> 这一课反其道而行：**源主机把一整串出端口号压进头栈，每台交换机 pop 一个端口号转发**——
> 转发路径完全由发送方指定，中间交换机不需要查路由表。这就是**源路由（source routing）**。
> 练习对应 `exercises/source_routing`。

## 学习目标

学完本课，你将能够：

1. 说出源路由与普通路由的区别（路径在包里 vs 路径在交换机表里）
2. 设计一个 2 字节的源路由头：`bos` + `port` 字段
3. 用 `hdr.srcRoutes.next` 和 `last.bos` 在 parser 里循环解析头栈
4. 用 `pop_front(1)` 在每台交换机"弹出一个端口号"
5. 用 `bos`（bottom of stack）判断"这是不是最后一跳"
6. 在最后一跳把 etherType 改回 IPv4，让接收主机正常解析
7. 独立补完 `source_routing.p4` 的全部 TODO 并跑通

---

## 1. 源路由思想：路径装在包里

### 1.1 对比普通路由

```
普通路由（第 4 课 basic）：
  源主机只知道"下一跳"，中间每台交换机自己查 ipv4_lpm 表决定往哪走
  路径 = 分散在各台交换机的转发表里

源路由（本课）：
  源主机把整条路径（一串出端口号）塞进包头
  每台交换机只做一件事：pop 一个端口号，从那个口发出去
  路径 = 装在包自己里面
```

### 1.2 为什么有用

- **零表项**：中间交换机完全不需要路由表（省 TCAM）。
- **可控路径**：发送方精确指定走哪条链路（避开拥塞/不可用链路）。
- **无状态转发**：交换机不记状态，只看包里"下一个要去的端口"。

### 1.3 打个比方

- 普通路由 = 每到一个路口问一次"往哪走"（每个路由器查自己的地图）。
- 源路由 = 出发前把整条路线写在小纸条上，每到一个路口撕下第一行照着走。

---

## 2. 头设计：`srcRoute_t`

### 2.1 定义（source_routing.p4:26-29）

```cpp
header srcRoute_t {
    bit<1>    bos;     // bottom of stack：是不是栈底（最后一个元素）
    bit<15>   port;    // 出端口号（15 位，够表示 0~32767 端口）
}
```

- 1 + 15 = **16 位 = 2 字节**，每个源路由"入口"只占 2 字节。
- **`port`**：这一跳要从哪个端口出去（填的是数据面端口号，如 1、2、3…）。
- **`bos`**（bottom of stack）：标记"我是不是路径上的最后一个"。最后一个元素 `bos=1`，
  前面的都是 `bos=0`。

### 2.2 放进 headers：头栈

```cpp
struct headers {
    ethernet_t           ethernet;
    srcRoute_t[MAX_HOPS] srcRoutes;    // 头栈：最多 9 个源路由入口
    ipv4_t               ipv4;
}
```

包的布局：

```text
[eth][srcRoute0][srcRoute1][srcRoute2]...[IPv4][UDP...]
  ↑  bos=0/port=A    ↑bos=0/port=B  ↑bos=1/port=C
  （先走 A）        （再走 B）      （最后一跳 C，然后到 IPv4）
```

> `bos=1` 在**路径最后一个**元素上（不是第一个！）——因为解析是从 eth 往后读，
> 读到 `bos=1` 就说明"源路由头到此结束，接下来是 IPv4"。

---

## 3. Parser：循环解析源路由栈

### 3.1 用 `bos` 判断栈结束（solution 第 77-83 行）

```cpp
state parse_ethernet {
    packet.extract(hdr.ethernet);
    transition select(hdr.ethernet.etherType) {
        TYPE_SRCROUTING: parse_srcRouting;    // 0x1234 → 有源路由头
        default: accept;
    }
}

state parse_srcRouting {
    packet.extract(hdr.srcRoutes.next);       // ① 解一个源路由入口
    transition select(hdr.srcRoutes.last.bos) {   // ② 看刚解的这个的 bos
        1: parse_ipv4;                        //    bos=1 → 是最后一个 → 解 IPv4
        default: parse_srcRouting;            //    bos=0 → 还有 → 继续解下一个（循环）
    }
}
```

### 3.2 关键语法点

- **`hdr.srcRoutes.next`**：头栈的"下一个待解元素"（第 6 课讲过），每次 extract 自动推进下标。
- **`hdr.srcRoutes.last.bos`**：`last` 指向**刚解析出的最后一个有效元素**。刚 extract 完
  `srcRoutes.next`，`last` 就是它，读它的 `bos` 判断"后面还有没有源路由头"。
- 循环结构：`bos=0` → 回到 `parse_srcRouting` 继续 extract 下一个；`bos=1` → 跳去 `parse_ipv4`。
- **和 MRI 的对比**：MRI 用 `mri.count` 决定循环次数；源路由用 `bos` 标志决定循环是否结束。

> 执行示例（路径 2 → 3 → 1）：
> ```
> extract(next) → srcRoutes[0] (bos=0) → last.bos=0 → 继续
> extract(next) → srcRoutes[1] (bos=0) → last.bos=0 → 继续
> extract(next) → srcRoutes[2] (bos=1) → last.bos=1 → parse_ipv4
> 结果：srcRoutes[0,1,2] 有效，其余 invalid
> ```

---

## 4. Ingress：逐跳 pop 端口转发

### 4.1 三个动作（TODO ①，对照 solution 第 114-125 行）

```cpp
action srcRoute_nhop() {
    standard_metadata.egress_spec = (bit<9>)hdr.srcRoutes[0].port;  // ① 出端口 = 栈顶的 port
    hdr.srcRoutes.pop_front(1);                                     // ② 弹出栈顶（这跳用完了）
}

action srcRoute_finish() {
    hdr.ethernet.etherType = TYPE_IPV4;   // 把以太网类型改回 0x0800
}

action update_ttl() {
    hdr.ipv4.ttl = hdr.ipv4.ttl - 1;      // 每跳 TTL-1（防环）
}
```

逐行解释：

| 动作 | 代码 | 干什么 |
|---|---|---|
| `srcRoute_nhop` | `egress_spec = srcRoutes[0].port` | 栈顶存的就是本跳要去的端口，直接设为出端口 |
| | `pop_front(1)` | **弹出栈顶**——用完了就删掉，下一台交换机看到的是"剩下路径" |
| `srcRoute_finish` | `etherType = TYPE_IPV4` | 源路由头全部用完时，把帧类型改回 IPv4，接收端才能正确解析 |
| `update_ttl` | `ttl = ttl - 1` | 每跳减一（和普通转发一致） |

### 4.2 `pop_front` vs `push_front`（和第 6 课对比）

| | `push_front(1)`（MRI） | `pop_front(1)`（源路由） |
|---|---|---|
| 方向 | 在**栈顶（0 号位）压入**一个新元素 | 从**栈顶（0 号位）弹出**一个元素 |
| 效果 | 新元素占 0 号位，旧的往后移 | 0 号位删除，后面的往前挪 |
| 用途 | 逐跳**添加**记录 | 逐跳**消费**路径 |

```
pop_front 前：srcRoutes = [2][3][1]
pop_front(1) 后：srcRoutes = [3][1]      ← 2 被弹掉了，下一跳用 3
```

### 4.3 apply 的控制流（TODO ②，对照 solution 第 127-139 行）

```cpp
apply {
    if (hdr.srcRoutes[0].isValid()) {          // ① 还有源路由要处理吗
        if (hdr.srcRoutes[0].bos == 1) {       // ② 栈顶是最后一个？
            srcRoute_finish();                 //    是 → 改 etherType 为 IPv4
        }
        srcRoute_nhop();                       // ③ 从栈顶取端口转发 + pop
        if (hdr.ipv4.isValid()) {              // ④ 有 IPv4 头就减 TTL
            update_ttl();
        }
    } else {
        drop();                                // ⑤ 没有源路由头 → 丢
    }
}
```

逻辑流程：

```
包进来
  ├─ srcRoutes[0] 无效 → drop（这不是源路由包）
  ├─ srcRoutes[0] 有效：
  │    栈顶 bos==1（最后一跳）→ 先 etherType 改回 IPv4
  │    取 srcRoutes[0].port 设为 egress_spec
  │    pop_front(1) 弹出栈顶
  │    IPv4 有效 → TTL-1
  → 从 egress_spec 发出
```

**为什么 `srcRoute_finish` 在 `srcRoute_nhop` 之前？** 最后一跳的交换机既要"从栈顶取端口发出"
（那跳的端口），又要"把 etherType 改回 IPv4"（因为出了这台交换机就是接收主机了）。
所以先改 etherType（deparser 会用新的值 emit），再取端口+pop。

### 4.4 逐跳模拟（路径 2 → 3 → 1，h1 发往 h2）

```
源主机 h1：包 = [eth][bos0/port2][bos0/port3][bos1/port1][IPv4]
第1台交换机 s_a：
  srcRoutes[0] = port2, bos=0 → egress_spec=2 → pop → 剩 [3][1]
  发出：[eth][bos0/port3][bos1/port1][IPv4]
第2台交换机 s_b：
  srcRoutes[0] = port3, bos=0 → egress_spec=3 → pop → 剩 [1]
  发出：[eth][bos1/port1][IPv4]
第3台交换机 s_c：
  srcRoutes[0] = port1, bos=1 → srcRoute_finish（etherType→0x0800）→ egress_spec=1 → pop → 剩 []
  发出：[eth][IPv4]   ← 源路由头全没了，就是一个普通 IPv4 包
接收主机 h2：看到 etherType=0x0800 → 正常解析 IP/UDP
```

> 注意：**路径在每台交换机处缩短一段**，最后一跳的交换机把源路由头"消费干净"，
> 接收端收到的就是一个普通 IPv4 包（第 6 课的 MRI 相反——头是越长越多）。

---

## 5. Deparser：emit 源路由栈

```cpp
control MyDeparser(packet_out packet, in headers hdr) {
    apply {
        packet.emit(hdr.ethernet);
        packet.emit(hdr.srcRoutes);   // 只 emit 栈里还 valid 的元素（没被 pop 完的）
        packet.emit(hdr.ipv4);
    }
}
```

- 源路由头是练习版里**已经写好的**（不需要 TODO）。
- `emit(hdr.srcRoutes)` 只输出**仍有效**的元素——被 `pop_front` 掉的不再 emit，所以路径逐跳变短。
- 顺序：eth → srcRoutes → ipv4，和 parser 解析顺序一致。

---

## 6. 控制面 / 发送端：`send.py` 怎么造源路由包

`exercises/source_routing/send.py` 用 scapy 构造：

```python
class SourceRoute(Packet):
    fields_desc = [ BitField("bos", 0, 1),   # 1 位
                    BitField("port", 0, 15)]  # 15 位

bind_layers(Ether, SourceRoute, type=0x1234)        # etherType=0x1234 → 源路由
bind_layers(SourceRoute, SourceRoute, bos=0)         # bos=0 → 下一个还是源路由
bind_layers(SourceRoute, IP, bos=1)                  # bos=1 → 下一个是 IPv4

# 交互输入端口路径，如 "2 3 2 2 1"
pkt = Ether(...) 
for p in s.split(" "):
    pkt = pkt / SourceRoute(bos=0, port=int(p))
pkt.getlayer(SourceRoute, i).bos = 1        # 最后一个设为 bos=1
pkt = pkt / IP(dst=addr) / UDP(...)         # 再接 IP 头
sendp(pkt, ...)
```

运行（**mininet 指令方式**，不用 xterm——send.py 是交互式的，用 `printf` 喂输入）：

```
mininet> h2 ./receive.py &                                          # ① 接收端后台跑（h2）
mininet> h1 bash -c 'printf "2 3 2 2 1\nq\n" | ./send.py 10.0.2.2'  # ② 发送端：喂端口序列
```

- `printf "2 3 2 2 1\nq\n"`：第一行是端口序列（发一个包），第二行 `q` 让 send.py 干净退出。
- 端口序列 `2 3 2 2 1` 对应 README 的路径：h1 → s1 → s2 → s3 → s1 → s2 → h2（中间有个环，TTL 会多减几次）。
- 想发别的路径，把引号里的 `2 3 2 2 1` 换成别的端口序列即可。
- **这完全不需要控制面表项**——路径在包里，交换机不用查表（对比 basic 的 runtime.json）。

---

## 7. 动手：补完 source_routing.p4 的 TODO

| TODO | 位置 | 要写什么 |
|---|---|---|
| ① parser | `parse_ethernet`（source_routing.p4:70-77） | 改成 `select(etherType)`：`TYPE_SRCROUTING → parse_srcRouting`，其余 `accept` |
| ② parser | `parse_srcRouting`（80-87） | `extract(srcRoutes.next)`；`select(last.bos)`：1→parse_ipv4，否则循环 |
| ③ ingress | `srcRoute_nhop`（118-124） | `egress_spec = (bit<9>)srcRoutes[0].port`；`pop_front(1)` |
| ④ ingress | `apply`（134-149） | `srcRoutes[0].isValid()` 内：bos==1→`srcRoute_finish()`；再 `srcRoute_nhop()`；IPv4 valid→`update_ttl()`；else `drop()` |

编译 + 运行：

```bash
cd /workspace/tutorials/exercises/source_routing
make run
```

进 mininet 后（**用 mininet 指令，不用 xterm**）：

```cpp
mininet> h2 ./receive.py &                                          # ① 接收端后台跑（h2）
mininet> h1 bash -c 'printf "2 3 2 2 1\nq\n" | ./send.py 10.0.2.2'  # ② 发送端：喂端口序列
```

- 第 ② 条：`printf "端口序列\nq\n" | ./send.py 目的IP` —— 用管道模拟 send.py 的交互输入
  （`q` 让它在发完包后正常退出）。
- 第 ③ 步验证：看 h2 的 receive.py 有没有打印、`h1 ./receive.py &` 反向再测一次
  （h2 发回 h1 时目的 IP 写 `10.0.1.1`）。
- 多跳/环路路径会多减 TTL：如 `2 3 2 2 1` 让 TTL 从 64 减到 59 再到达。

---

## 8. 本课小结

- **源路由思想**：路径 = 一串出端口号，装在包自己的头栈里；交换机只 pop 不查表。
- **头设计**：`srcRoute_t { bos(1) + port(15) }`，2 字节一个入口，`bos` 标记栈底。
- **parser 循环**：`extract(srcRoutes.next)` + `select(last.bos)`——bos=0 继续解，bos=1 转 IPv4。
- **逐跳消费**：`egress_spec = srcRoutes[0].port` + `pop_front(1)`，路径逐跳变短。
- **最后一跳**：`srcRoute_finish()` 把 etherType 改回 IPv4，接收端才能正常解析。
- **和 MRI 对比**：MRI 逐跳 **push**（头变长）；源路由逐跳 **pop**（头变短）。

## 9. 思考题

1. 源路由和普通路由（ipv4_lpm）分别把"路径信息"存在哪？各有什么优缺点？
2. `bos` 为什么标在**最后一个**源路由头上，而不是第一个？parser 靠它做什么？
3. `hdr.srcRoutes.last.bos` 和 `hdr.srcRoutes[0].bos` 有什么不同？分别在哪个阶段用？
4. 如果路径是 `2 3 1`，第 2 台交换机执行 `srcRoute_nhop` 后，栈里还剩什么？
5. 为什么不 pop 完所有源路由头就发到主机，主机能解析吗？（提示：etherType 还是 0x1234）
6. `pop_front` 和 `push_front` 对头栈长度的影响分别是什么？
7. 路径超过 `MAX_HOPS=9` 会怎样？怎么改进？

## 10. 延伸阅读

- `exercises/source_routing/README.md`（实验步骤、send.py 交互）
- 第 6 课《头栈与带内网络遥测》（头栈基础：`.next`/`.last`/`push_front`/`pop_front`）
- P4-16 规范头栈章节：`https://p4.org/p4-spec/docs/P4-16-v1.2.3.html`
- 下一个：第 8 课《自定义协议与哈希 —— P4 计算器与负载均衡》（`exercises/calc`）

---

*下一课预告：第 8 课《自定义协议与哈希》。源路由是"用头栈指定路径"，下一课要用
自定义协议头做"在数据面算算术"（P4 计算器），再用哈希做负载均衡。*