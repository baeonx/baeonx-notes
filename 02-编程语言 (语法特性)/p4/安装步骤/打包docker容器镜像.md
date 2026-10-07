## 📦 打包 Docker 容器为镜像

### 第一步：确认容器状态

```bash
# 查看容器是否在运行
docker ps -a | grep p4-dev
```

如果容器已停止，先启动它（打包前建议确保容器处于运行或停止状态都可以，但运行状态能保存更多运行时信息）：

```bash
docker start p4-dev
```

------

### 第二步：将容器打包为镜像

```bash
docker commit p4-dev p4-ubuntu-full
```

- `p4-dev`：你的容器名
- `p4-ubuntu-full`：新镜像名（可以自定义）

**如果希望添加版本标签**：

```bash
docker commit p4-dev p4-ubuntu-full:v1.0
```

------

### 第三步：验证镜像

```bash
# 查看镜像列表
docker images | grep p4-ubuntu-full

# 应该看到类似输出：
# p4-ubuntu-full   latest    xxxxx   10 seconds ago   2.3GB
```

------

### 第四步：测试新镜像

```bash
# 用新镜像启动测试容器
docker run -it --rm --privileged p4-ubuntu-full:v1.0 bash

# 在容器内验证环境
p4c --version
simple_switch --version
exit

docker run -d \
  -v $(pwd):/workspace \
  -w /workspace \
  --privileged \
  --name p4-dev \
  -p 2222:22 \
  --add-host host.docker.internal:172.18.0.1 \
  p4-ubuntu-full:v1.0 \
  tail -f /dev/null
```

------

## 🚀 其他常用打包命令

| 操作                          | 命令                                                         |
| :---------------------------- | :----------------------------------------------------------- |
| 打包并添加版本标签            | `docker commit -a "你的名字" -m "添加描述信息" p4-dev p4-ubuntu-full:v1.0` |
| 查看镜像详情                  | `docker inspect p4-ubuntu-full`                              |
| 导出为 tar 文件（备份用）     | `docker save -o p4-ubuntu-full.tar p4-ubuntu-full`           |
| 从 tar 文件导入               | `docker load -i p4-ubuntu-full.tar`                          |
| 删除镜像                      | `docker rmi p4-ubuntu-full`                                  |
| 推送到 Docker Hub（需要登录） | `docker tag p4-ubuntu-full 你的用户名/p4-ubuntu-full && docker push 你的用户名/p4-ubuntu-full` |

------

## 📌 镜像大小说明

这个镜像大约 **2-3GB**，因为包含了：

- Ubuntu 20.04 基础系统
- 所有编译依赖和工具
- BMv2 和 P4C 编译产物

如果想减小体积，可以用 `docker export` 导出为轻量级 tar（但会丢失历史记录和元数据）。

------

执行 `docker commit p4-dev p4-ubuntu-full` 和 `docker images | grep p4-ubuntu-full`，把结果发给我确认！







## 🔍 查看所有代理配置

### 1️⃣ 查看环境变量中的代理

```bash
env | grep -i proxy
```

------

### 2️⃣ 查看 APT 代理配置

```bash
cat /etc/apt/apt.conf.d/95proxy
```

------

### 3️⃣ 查看 Git 代理配置

```bash
git config --global --list | grep proxy
```

------

### 4️⃣ 查看 `~/.bashrc` 中的代理设置

```bash
cat ~/.bashrc | grep -i proxy
```

------

### 5️⃣ 查看 Docker 容器启动时传入的环境变量

```bash
# 在宿主机执行
docker inspect p4-dev | grep -A 10 "Env"
```

------

## 📌 一键查看所有代理

```bash
echo "=== 环境变量 ===" && env | grep -i proxy && \
echo "=== APT 代理 ===" && cat /etc/apt/apt.conf.d/95proxy 2>/dev/null && \
echo "=== Git 代理 ===" && git config --global --list 2>/dev/null | grep proxy && \
echo "=== bashrc 代理 ===" && cat ~/.bashrc 2>/dev/null | grep -i proxy
```

------

执行 `env | grep -i proxy`，把结果发给我！