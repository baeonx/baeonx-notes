# DPDK 学习路线（从基础到进阶）

> 面向对象：具备 C 语言与 Linux 基础的网络/数据平面学习者。若你在学 P4 数据平面，DPDK 的"用户态数据面"理念与 P4 的 match-action 模型相通，可互为印证。
>
> 当前版本（2026 年）：DPDK **25.11 LTS** 与 **26.07** 为活跃版本，文档以官网为准。

---

## 目录

1. 第一阶段：前置知识（基础中的基础）
2. 第二阶段：核心概念（建立心智模型）
3. 第三阶段：核心库实战（动手写代码）
4. 第四阶段：示例应用实战
5. 第五阶段：进阶方向
6. 第六阶段：与 P4 结合（可选衔接）
7. 学习资料汇总
8. 学习建议与避坑

---

## 第一阶段：前置知识

| 知识 | 要点 | 周期 |
| :--- | :--- | :--- |
| C 语言 | 指针、结构体、内存管理（malloc/free）、多线程（pthread） | 2-3 周 |
| Linux 网络 | TCP/IP 协议栈、网卡收发包流程、中断 vs 轮询、socket | 1 周 |
| Linux 系统 | 内核模块、`/proc`、hugepage 大页、CPU 亲和性、`taskset` | 1 周 |
| 编译工具 | GCC、Make、Meson（DPDK 用 meson/ninja 构建） | 可边学边用 |

> **验收标准：** 能用 C 写一个基于 socket 的简单 TCP 收发程序；能看懂 `/proc/meminfo` 中的 HugePages_Total。

📚 **本阶段资料：**

| 主题 | 资料 | 地址 |
| :--- | :--- | :--- |
| C 语言 | 菜鸟教程 C 语言 | https://www.runoob.com/cprogramming/c-tutorial.html |
| C 语言 | 《C Primer Plus》（经典书籍） | 各电商/图书馆有售 |
| Linux 系统 | 《鸟哥的 Linux 私房菜》（免费在线） | https://linux.vbird.org/ |
| TCP/IP | 《TCP/IP 详解 卷1：协议》 | 经典书籍 |
| 大页内存/系统要求 | DPDK 官方 Getting Started · 系统要求 | https://doc.dpdk.org/guides/linux_gsg/sys_reqs.html |

---

## 第二阶段：核心概念（先建立心智模型）

DPDK 的诞生源于"内核协议栈太慢"：每次收包要经过中断 → 内核 → 拷贝 → 系统调用 → 应用，性能受限。DPDK 的核心思想是 **完全绕开内核，让应用直接操作网卡**。

| 概念 | 一句话理解 | 为什么重要 |
| :--- | :--- | :--- |
| **用户态驱动** | 网卡由用户态应用直接管理，不经过内核 | 去掉系统调用与拷贝开销 |
| **PMD（轮询模式驱动）** | 应用忙轮询网卡队列，而非中断通知 | 避免中断上下文切换 |
| **EAL（环境抽象层）** | 初始化环境：管理线程(lcore)、内存、设备 | 所有程序第一步 |
| **Hugepage（大页内存）** | 用 2MB/1GB 大页避免 TLB 抖动 | 高性能的关键底座 |
| **Ring（无锁队列）** | 多核间传递 mbuf 的无锁环形队列 | 零锁并发的基础 |
| **Mbuf（报文缓冲区）** | 承载一个报文的缓冲区结构 | 所有包处理的对象 |
| **Mempool（内存池）** | 预分配 mbuf 对象池，复用避免频繁分配 | 高吞吐的保障 |
| **NUMA** | CPU 与内存的物理亲缘，跨节点访问慢 | 多核部署必须考虑 |
| **零拷贝** | mbuf 直接传递，避免内核-用户态拷贝 | 性能核心卖点 |
| **Batch（批量收发）** | `rx_burst/tx_burst` 一次收/发一批包 | 摊销循环开销 |

> **关键认知：** DPDK 不是"加速内核"，而是"绕过内核"。代价是：丢掉了内核协议栈的全部功能（TCP 栈、路由、防火墙），这些需要你自己用 DPDK 库实现。

📚 **本阶段资料：**

| 资料 | 说明 | 地址 |
| :--- | :--- | :--- |
| Programmer's Guide · 简介/概述 | 官方架构总览（必读） | https://doc.dpdk.org/guides/prog_guide/intro.html |
| Programmer's Guide · 全本 | 官方编程指南（核心文档） | https://doc.dpdk.org/guides/prog_guide/ |
| DPDK 文档中文翻译 · 编程指南 | 概念章节的中译 | https://dpdk-docs.readthedocs.io/en/latest/prog_guide/ |
| 书：《深入浅出DPDK》 | 中文经典，讲透架构与原理 | 电商/图书馆 |
| DPDK Hands-on · 概念导读 | 中文实战教程的概念部分 | https://haolipeng.github.io/dpdk-hands-on/ |
| DPDK 初学者入门必读（谢先斌博客） | 术语速览 | https://www.xiexianbin.cn/sdn/dpdk/must-read-for-dpdk-beginner/ |

---

## 第三阶段：核心库实战（动手写代码）

这是学习主战场，**每个库都要跑官方 sample 并自己改一版**。建议顺序：

| 步骤 | 库/主题 | 关键 API | 官方示例 |
| :--- | :--- | :--- | :--- |
| 1 | **EAL 初始化** | `rte_eal_init()`、`RTE_LCORE_FOREACH_WORKER` | `helloworld` |
| 2 | **Mbuf** | `rte_pktmbuf_alloc/free`、`rte_pktmbuf_mtod` | `skeleton` |
| 3 | **Mempool** | `rte_mempool_create`、`rte_pktmbuf_pool_create` | `skeleton` |
| 4 | **Ring** | `rte_ring_enqueue/dequeue`、SP/SC、MP/MC 模式 | `l2fwd` 的内部 |
| 5 | **网卡初始化/PMD** | `rte_eth_dev_configure`、`rte_eth_rx_queue_setup` | `skeleton`、`l2fwd` |
| 6 | **收发包** | `rte_eth_rx_burst` / `rte_eth_tx_burst` | `l2fwd`、`testpmd` |
| 7 | **LPM（路由表）** | `rte_lpm_add/lookup` | `l3fwd` |
| 8 | **Hash（哈希表）** | `rte_hash_add/lookup` | `l3fwd`、`flow_classify` |
| 9 | **ACL（访问控制）** | `rte_acl_*` | `acl` |
| 10 | **Timer / Distribute / KNI / QoS** | 进阶库，按需学 | 各 sample |

> **建议每周一个主题**：先读官方 sample 源码 → 编译运行 → 修改参数/加打印 → 换一种实现验证理解。

📚 **本阶段资料（各库官方文档）：**

| 库 | 官方文档地址 |
| :--- | :--- |
| EAL 环境抽象层 | https://doc.dpdk.org/guides/prog_guide/env_abstraction_layer.html |
| Mbuf | https://doc.dpdk.org/guides/prog_guide/mbuf_lib.html |
| Mempool | https://doc.dpdk.org/guides/prog_guide/mempool_lib.html |
| Ring | https://doc.dpdk.org/guides/prog_guide/ring_lib.html |
| PMD / 网卡轮询驱动 | https://doc.dpdk.org/guides/prog_guide/poll_mode_drv.html |
| LPM 路由表 | https://doc.dpdk.org/guides/prog_guide/lpm_lib.html |
| Hash 哈希表 | https://doc.dpdk.org/guides/prog_guide/hash_lib.html |
| ACL 访问控制 | https://doc.dpdk.org/guides/prog_guide/acl_lib.html |
| Timer / KNI / QoS | https://doc.dpdk.org/guides/prog_guide/ 中对应章节 |
| 中文手写教程（逐库代码） | DPDK Hands-on：https://github.com/haolipeng/dpdk-hands-on |

---

## 第四阶段：示例应用实战

把官方示例当成"必做作业"：

1. **helloworld** —— 理解 EAL 与多核（⭐）
2. **skeleton** —— 最小的收发包框架（⭐）
3. **l2fwd** —— 二层转发，理解队列与收发包循环（⭐⭐）
4. **l3fwd** —— 三层转发，LPM 查表（⭐⭐⭐）
5. **testpmd** —— 交互式网卡测试工具，**必须会用**（发流/收流/看统计）
6. 自己动手写：
   - 抓包器（rx_burst + 协议打印）
   - 简单以太网帧转发
   - 带 LPM 路由表的转发器
   - 五元组 Hash 查表 + 计数
   - 多队列 + 多核（每个 lcore 绑定一个队列）

> **验收标准：** 不看官方源码，能独立写出一个基于 DPDK 的 `l2fwd` 等价程序。

📚 **本阶段资料（官方示例应用文档 + 中文教程）：**

| 示例 | 官方文档 | 中文教程 |
| :--- | :--- | :--- |
| 示例应用总目录 | https://doc.dpdk.org/guides/sample_app_ug/ | — |
| helloworld | https://doc.dpdk.org/guides/sample_app_ug/helloworld.html | DPDK Hands-on Lesson 1 |
| skeleton | https://doc.dpdk.org/guides/sample_app_ug/skeleton.html | — |
| l2fwd | https://doc.dpdk.org/guides/sample_app_ug/l2_forward_real_virtual.html | DPDK Hands-on Lesson 3/4 |
| l3fwd | https://doc.dpdk.org/guides/sample_app_ug/l3_forward.html | — |
| testpmd 使用指南 | https://doc.dpdk.org/guides/testpmd_app_ug/ | DPDK Hands-on Lesson 13+ |
| DPDK Hands-on 中文教程 | 全课程导航 | https://haolipeng.github.io/dpdk-hands-on/ |

---

## 第五阶段：进阶方向（按兴趣选）

### 5.1 性能优化
- 批量处理与突发长度（`rx_burst` 一次 32/64 个）
- SIMD 向量指令（DPDK 24.x 起在 x86 引入向量处理优化）
- mbuf 复用与内存布局（缓存行对齐、prefetch 预取）
- 网卡多队列（RSS）、flow director
- 大页分配策略与 NUMA 绑核

### 5.2 多进程架构
- primary/secondary 进程模型（`--proc-type`）
- 共享内存与 `rte_ring` 跨进程通信
- 典型方案：数据面进程 + 控制面进程分离

### 5.3 生态与技术栈
| 技术 | 说明 | 学习重点 |
| :--- | :--- | :--- |
| **OVS-DPDK** | Open vSwitch 的 DPDK 加速版 | 虚拟交换机、vhost-user |
| **VPP（FD.io）** | Cisco 开源的高性能数据面框架 | 数据图节点、插件模型 |
| **DPDK 虚拟化** | vhost/virtio、vDPA | 虚拟机高性能网卡 |
| **Crypto/安全** | cryptodev、ipsec-secgw | 加解密卸载 |
| **智能网卡/卸载** | 流表卸载、DPDK Pipeline | 硬件加速 |

### 5.4 性能测量与调优
- `rte_eth_stats`、`rte_ring` 统计
- `testpmd` 的 `show port stats`
- 结合 `perf`、`pktgen`、`Moongen` 做基准测试

📚 **本阶段资料：**

| 主题 | 资料 | 地址 |
| :--- | :--- | :--- |
| 性能优化总览 | Programmer's Guide 性能/调优相关章节 | https://doc.dpdk.org/guides/prog_guide/ |
| 多进程支持 | Programmer's Guide · Multi-process | https://doc.dpdk.org/guides/prog_guide/multi_proc_support.html |
| OVS-DPDK | Open vSwitch 官方文档 | https://docs.openvswitch.org/ |
| VPP / FD.io | FD.io 官网与文档 | https://fd.io/ · https://wiki.fd.io/ |
| DPDK 虚拟化（vhost） | Programmer's Guide · vhost | https://doc.dpdk.org/guides/prog_guide/vhost_lib.html |
| 流量工具 | Pktgen（DPDK 版） | https://pktgen-dpdk.rtfd.io/ |
| 基准测试 | Moongen（高速流量发生器） | https://github.com/emmericp/MoonGen |

---

## 第六阶段：与 P4 结合（可选，衔接你正在学的 P4）

DPDK 与 P4 是两种不同的数据面技术，但可互补：

| 对比项 | P4（BMv2/芯片） | DPDK |
| :--- | :--- | :--- |
| 抽象层次 | 高级语言描述流水线 | C 库直接写数据面 |
| 表/匹配 | match-action 表（编译器生成） | LPM/Hash/ACL 库手工实现 |
| 后端目标 | 硬件芯片 / BMv2 仿真 | x86 通用 CPU |

衔接路径：
1. 用 DPDK 跑通 `testpmd` 与 `l2fwd`，理解"用户态数据面"；
2. 使用 **p4c 的 DPDK 后端 `p4c-dpdk`**（你环境已装于 `/usr/local/bin/p4c-dpdk`）：
   ```bash
   p4c-dpdk --p4v 1.2 --p4runtime-files ... program.p4
   ```
3. 把 P4 程序编译到 DPDK 目标，对比 BMv2 与 DPDK 的流水线差异，体会"P4 一次编写、多目标部署"。

📚 **本阶段资料：**

| 资料 | 说明 | 地址 |
| :--- | :--- | :--- |
| p4c 官方仓库 | p4c 编译器（含 DPDK 后端文档与测试） | https://github.com/p4lang/p4c |
| P4 官网 | P4 语言规范、教程、论文 | https://p4.org/ |
| P4 语言规范 | 官方 PDF 规范 | https://p4.org/p4-spec/ |
| P4 学习笔记 | 你本机的 `P4基础语法.md` 与 `P4内置函数从零基础到进阶教程.md` | 本地文件 |
| DPDK + P4 讨论 | p4c 仓库 issues 中 dpdk backend 相关 | https://github.com/p4lang/p4c/issues |

---

## 学习资料汇总

### 官方文档（最权威）
| 文档 | 用途 | 地址 |
| :--- | :--- | :--- |
| 官网 | 下载、新闻、版本 | https://www.dpdk.org/ |
| Getting Started Guide | 安装编译、运行 | https://doc.dpdk.org/guides/linux_gsg/ |
| Programmer's Guide | 编程指南（核心必读） | https://doc.dpdk.org/guides/prog_guide/ |
| API Reference | 函数/结构体速查 | https://doc.dpdk.org/api/ |
| Sample Applications | 示例应用详解 | https://doc.dpdk.org/guides/sample_app_ug/ |
| Release Notes | 版本变更 | https://doc.dpdk.org/guides/rel_notes/ |

### 中文资料
| 资源 | 说明 | 地址 |
| :--- | :--- | :--- |
| **DPDK Hands-on** | 强烈推荐，零基础→进阶的中文实战教程（14+ 课，含代码+讲解） | 教程 https://haolipeng.github.io/dpdk-hands-on/ · 源码 https://github.com/haolipeng/dpdk-hands-on |
| DPDK 文档中文翻译 | 入门指南/编程指南的中译 | https://dpdk-docs.readthedocs.io/ |
| 谢先斌博客 DPDK 系列 | 术语、源码安装、testpmd 入门 | https://www.xiexianbin.cn/sdn/dpdk/ |

### 书籍
- 《深入浅出DPDK》—— 中文经典，讲透架构与原理，适合先建立整体认知
- 《DPDK 应用基础教程》—— 偏实践操作
- 英文：DPDK 官方 Programmer's Guide 即是最好的教材；Linux Plumbers 大会 DPDK 议题可作拓展阅读

### 视频与社区
- 哔哩哔哩搜索「DPDK 入门 / DPDK 教程」，配合实操视频效率更高
- 官方邮件列表：`users@dpdk.org`（提问前先查 archive）
- GitHub：https://github.com/DPDK/dpdk
- DPDK Blog：https://www.dpdk.org/blog/

---

## 学习建议与避坑

1. **别只看不写**：DPDK 是"跑起来才懂"的技术，每个库都要编译运行 + 改造。
2. **官方示例是最好的老师**：先抄后改，再看源码库实现。
3. **没有真实网卡也能学**：早期用 DPDK 自带的**软件驱动**（`net_ring`、`net_null`、`net_pcap`、`net_tap`）即可在本机跑通大部分示例；有网卡后再上 PMD。
4. **注意版本差异**：DPDK 各版本 API 变化较大，教程可能与新版不兼容，优先看对应版本的官方文档。
5. **性能是"量"出来的**：用 `testpmd`/`pktgen` 测吞吐，用数据说话，不要凭感觉优化。
6. **大页和绑核是第一步**：`--huge-dir`、`-l`/`-c`、`--socket-mem` 这些 EAL 参数必须吃透。
7. **调试手段**：`--log-level=8` 看详细日志；`--no-huge` 便于调试；`gdb` 可以跟用户态代码。
8. **与 P4 相互印证**：DPDK 的 LPM/Hash 就是 P4 里 table 的 CPU 实现；P4 的 deparser 类似 DPDK 的 `rte_pktmbuf` 组装。两者一起学，理解更立体。