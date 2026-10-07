## 🐳 Docker + P4 开发环境完整安装指南（压缩包版）

### 📋 前置条件

- WSL 或 Linux 系统
- 已安装 Docker
- 有网络代理（用于加速下载，可选）

------

## 第一步：创建基础镜像

### 1.1 创建工作目录

```bash
mkdir -p ~/P4 && cd ~/P4
```



### 1.2 创建 Dockerfile

```bash
cat > Dockerfile << 'EOF'
FROM ubuntu:20.04

ENV DEBIAN_FRONTEND=noninteractive

# 安装基础编译工具和依赖
RUN apt-get update && apt-get install -y \
    wget git cmake g++ python3 python3-pip python3-dev \
    autoconf automake libtool \
    libboost-dev libboost-filesystem-dev libboost-iostreams-dev \
    libboost-program-options-dev libboost-thread-dev libboost-system-dev \
    libgmp-dev libpcap-dev libpcre3-dev flex bison \
    libssl-dev libedit-dev pkg-config \
    tcpdump vim net-tools iproute2 \
    libthrift-dev thrift-compiler \
    libnanomsg-dev libjsoncpp-dev \
    libevent-dev ccache \
    protobuf-compiler \
    && rm -rf /var/lib/apt/lists/*

# 安装 Python 依赖
RUN pip3 install scapy thrift

WORKDIR /workspace
CMD ["/bin/bash"]
EOF
```



### 1.3 构建基础镜像

```bash
docker build -t p4-ubuntu-base .
```



------

## 第二步：创建持久容器

### 2.1 创建启动脚本

```bash
cat > start.sh << 'EOF'
#!/bin/bash
docker run -d \
  -v $(pwd):/workspace \
  -w /workspace \
  --privileged \
  --name p4-dev \
  -e HTTP_PROXY=http://host.docker.internal:7890 \
  -e HTTPS_PROXY=http://host.docker.internal:7890 \
  -e http_proxy=http://host.docker.internal:7890 \
  -e https_proxy=http://host.docker.internal:7890 \
  p4-ubuntu-base \
  tail -f /dev/null
EOF

chmod +x start.sh
```



### 2.2 启动容器

```bash
./start.sh
```



### 2.3 进入容器

```bash
docker exec -it p4-dev bash
```



**此后所有命令都在容器内执行**（提示符为 `root@xxxxx:/workspace#`）。

------

## 第三步：编译安装 BMv2（压缩包版）

### 3.1 下载并解压 BMv2 源码

```bash
cd /root

# 下载 BMv2 压缩包
curl -L -o bmv2.tar.gz \
  https://github.com/p4lang/behavioral-model/archive/refs/heads/main.tar.gz

# 解压
tar -xzf bmv2.tar.gz
mv behavioral-model-main behavioral-model
cd behavioral-model
```



### 3.2 编译安装

```bash
# 去掉脚本中的 sudo（容器内是 root）
sed -i 's/sudo //g' install_deps.sh
./install_deps.sh

apt-get install -y libgrpc++-dev libgrpc-dev protobuf-compiler-grpc
./autogen.sh
./configure --enable-grpc

# 编译安装
./autogen.sh
./configure
make -j1
make install
ldconfig

# 验证
simple_switch --version
simple_switch_grpc --version
```



**预期输出**：`1.15.5-xxxxxxx`

------

## 第四步：编译安装 P4C（压缩包版）

### 4.1 下载并解压 P4C 源码

```bash
cd /root

# 下载 P4C 压缩包
curl -L -o p4c.tar.gz \
  https://github.com/p4lang/p4c/archive/refs/heads/main.tar.gz

# 解压
tar -xzf p4c.tar.gz
mv p4c-main p4c
cd p4c
```



### 4.2 配置 Git 代理（重要！）

```bash
# 让 cmake 下载依赖时使用代理
git config --global http.proxy http://host.docker.internal:7890
git config --global https.proxy http://host.docker.internal:7890
```



### 4.3 编译安装

```bash
mkdir build && cd build
cmake ..
make -j1
make install

# 验证
p4c --version
```

**预期输出**：`p4c 1.2.5.16 (SHA: BUILD: Release)`

------

## 第五步：测试运行

### 5.1 编写测试程序

```bash
cd /workspace

cat > test.p4 << 'EOF'
#include <core.p4>
#include <v1model.p4>

typedef bit<9> egressSpec_t;
typedef bit<48> macAddr_t;
typedef bit<32> ip4Addr_t;

header ethernet_t {
    macAddr_t dstAddr;
    macAddr_t srcAddr;
    bit<16>   etherType;
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

struct headers {
    ethernet_t ethernet;
    ipv4_t     ipv4;
}

struct metadata {}

parser MyParser(packet_in packet, out headers hdr, inout metadata meta, inout standard_metadata_t standard_metadata) {
    state start { transition parse_ethernet; }
    state parse_ethernet {
        packet.extract(hdr.ethernet);
        transition select(hdr.ethernet.etherType) {
            0x0800: parse_ipv4;
            default: accept;
        }
    }
    state parse_ipv4 {
        packet.extract(hdr.ipv4);
        transition accept;
    }
}

control MyIngress(inout headers hdr, inout metadata meta, inout standard_metadata_t standard_metadata) {
    action drop() {
        mark_to_drop(standard_metadata);
    }
    action ipv4_forward(macAddr_t dstAddr, egressSpec_t port) {
        standard_metadata.egress_spec = port;
        hdr.ethernet.srcAddr = hdr.ethernet.dstAddr;
        hdr.ethernet.dstAddr = dstAddr;
        hdr.ipv4.ttl = hdr.ipv4.ttl - 1;
    }
    table ipv4_lpm {
        key = { hdr.ipv4.dstAddr: lpm; }
        actions = { ipv4_forward; drop; NoAction; }
        size = 1024;
        default_action = drop();
    }
    apply {
        if (hdr.ipv4.isValid()) {
            ipv4_lpm.apply();
        }
    }
}

control MyEgress(inout headers hdr, inout metadata meta, inout standard_metadata_t standard_metadata) {
    apply { }
}

control MyDeparser(packet_out packet, in headers hdr) {
    apply {
        packet.emit(hdr.ethernet);
        packet.emit(hdr.ipv4);
    }
}

control MyVerifyChecksum(inout headers hdr, inout metadata meta) { apply { } }
control MyComputeChecksum(inout headers hdr, inout metadata meta) { apply { } }

V1Switch(
    MyParser(),
    MyVerifyChecksum(),
    MyIngress(),
    MyEgress(),
    MyComputeChecksum(),
    MyDeparser()
) main;
EOF
```



### 5.2 编译 P4 程序

```bash
p4c -b bmv2 -o build test.p4
ls -la build/
# 应该看到 test.json
```



### 5.3 运行测试

**终端1 - 启动交换机**（保持运行）：

```bash
# 创建虚拟网卡
ip link add veth0 type veth peer name veth1
ip link add veth2 type veth peer name veth3
ip link set dev veth0 up
ip link set dev veth1 up
ip link set dev veth2 up
ip link set dev veth3 up

# 启动交换机（带日志）
simple_switch -i 0@veth0 -i 1@veth2 --log-console build/test.json
```



**终端2 - 配置流表并发送数据包**（新 WSL 窗口）：

```bash
# 进入容器
docker exec -it p4-dev bash

# 配置流表
simple_switch_CLI --thrift-port 9090 << 'CLI'
table_add ipv4_lpm ipv4_forward 10.0.1.1/32 => 00:00:00:00:00:02 1
CLI

# 发送测试包
python3 << 'PYTHON'
from scapy.all import Ether, IP, sendp
pkt = Ether(src="00:00:00:00:00:01", dst="ff:ff:ff:ff:ff:ff", type=0x0800) / IP(src="192.168.1.1", dst="10.0.1.1")
sendp(pkt, iface="veth1", verbose=True)
print("✅ IPv4 数据包已发送")
PYTHON
```



------

## 第六步：保存镜像（可选）

如果希望以后不用重新编译，保存当前容器为镜像：

```bash
# 在宿主机执行
docker commit p4-dev p4-ubuntu-full
```



以后直接用这个镜像启动容器：

```bash
docker run -it --rm --privileged p4-ubuntu-full bash
```



------

## 📋 常用命令速查

| 操作         | 命令                                  |
| :----------- | :------------------------------------ |
| 启动容器     | `docker start p4-dev`                 |
| 进入容器     | `docker exec -it p4-dev bash`         |
| 停止容器     | `docker stop p4-dev`                  |
| 删除容器     | `docker rm -f p4-dev`                 |
| 查看容器状态 | `docker ps -a | grep p4-dev`          |
| 保存镜像     | `docker commit p4-dev p4-ubuntu-full` |

------

## ✅ 验证清单

安装完成后，确认以下命令都能正常执行：

```bash
p4c --version          # p4c 1.2.5.16
simple_switch --version # 1.15.5-xxxxxxx
```



------

## ⚠️ 注意事项

1. **代理配置**：如果你没有代理，可以去掉 `start.sh` 中的 `-e HTTP_PROXY` 相关行。
2. **`host.docker.internal`**：只在 Docker Desktop（macOS/Windows）中有效。如果你在 Linux 上运行 Docker，需要将其改为 `172.17.0.1`。
3. **`socat` 转发**：如果你的 Clash 代理只监听 `127.0.0.1:7890`，容器内无法直接使用，需要先启动 `socat` 转发。

恭喜！现在你有了一个完整的 P4 开发环境！**🎉**





