# 📚 Docker 容器 SSH 配置 + VS Code 远程连接完整教程

------

## 📋 前提条件

- Docker 容器 `p4-dev` 已创建并运行
- 宿主机端口 `2222` 已映射到容器 `22`（创建容器时加 `-p 2222:22`）

------

# 第一部分：容器内安装 SSH

## 1.1 进入容器

```bash
docker exec -it p4-dev bash
```

------

## 1.2 安装 SSH 服务

```bash
# 更新软件源
apt-get update

# 安装 SSH
apt-get install -y openssh-server sudo
```

------

## 1.3 配置 SSH

```bash
# 设置 root 密码（登录用）
echo "root:123456" | chpasswd

# 允许 root 用户密码登录
sed -i 's/#PermitRootLogin prohibit-password/PermitRootLogin yes/' /etc/ssh/sshd_config
sed -i 's/#PasswordAuthentication yes/PasswordAuthentication yes/' /etc/ssh/sshd_config
```

**这两条命令的作用**：

| 命令                         | 作用                                 |
| :--------------------------- | :----------------------------------- |
| `PermitRootLogin yes`        | 允许 root 用户通过 SSH 登录          |
| `PasswordAuthentication yes` | 允许使用密码登录（而不是只能用密钥） |

------

## 1.4 启动 SSH 服务

```bash
service ssh start
```

------

## 1.5 验证 SSH 是否运行

```bash
ss -tlnp | grep 22
```

**预期输出**：

```text
LISTEN    0    128    0.0.0.0:22    0.0.0.0:*    users:(("sshd",pid=xxx,fd=3))
```



------

# 第二部分：宿主机验证

## 2.1 检查端口映射

```bash
ss -tlnp | grep 2222
```

**预期输出**：看到 `0.0.0.0:2222` 在监听。

------

## 2.2 测试 SSH 连接（宿主机）

```bash
ssh -p 2222 root@localhost
```

输入密码 `123456`，应该能成功进入容器。

------

# 第三部分：VS Code Remote-SSH 配置

## 3.1 安装 Remote-SSH 扩展

在 VS Code 中搜索并安装 **Remote - SSH**（作者：Microsoft）。

------

## 3.2 配置 SSH 连接

按 `F1` → 输入 **Remote-SSH: Open SSH Configuration File** → 选择 `~/.ssh/config`。

添加以下配置：

```bash
Host p4-dev
  HostName localhost
  User root
  Port 2222
  PasswordAuthentication yes
```

**保存文件**（`Ctrl+S`）。

------

## 3.3 连接容器

按 `F1` → **Remote-SSH: Connect to Host...** → 选择 `p4-dev`。

输入密码 `123456`。

------

## 3.4 连接成功标志

- VS Code 左下角显示 `SSH: p4-dev`
- 终端显示 `root@容器ID:/workspace#`

------

# 第四部分：常见问题排查

## ❌ 连接超时

```bash
# 检查容器是否运行
docker ps | grep p4-dev

# 检查端口映射
ss -tlnp | grep 2222
```

------

## ❌ Permission denied (publickey)

```bash
# 检查 SSH 配置是否正确
docker exec -it p4-dev cat /etc/ssh/sshd_config | grep -E "PermitRootLogin|PasswordAuthentication"
```

应该显示：

```text
PermitRootLogin yes
PasswordAuthentication yes
```

------

## ❌ Connection refused

```bash
# 检查 SSH 是否运行
docker exec -it p4-dev service ssh status

# 如果没运行，启动它
docker exec -it p4-dev service ssh start
```

------

## ❌ 端口已被占用

```bash
# 查看 2222 端口被谁占用
sudo lsof -i :2222

# 换一个端口（比如 2223）重新创建容器
docker run -d -p 2223:22 ...
```

------

# 第五部分：一键安装脚本

如果不想手动输入，可以直接运行：

```bash
docker exec -it p4-dev bash -c "
apt-get update && \
apt-get install -y openssh-server sudo && \
echo 'root:123456' | chpasswd && \
sed -i 's/#PermitRootLogin prohibit-password/PermitRootLogin yes/' /etc/ssh/sshd_config && \
sed -i 's/#PasswordAuthentication yes/PasswordAuthentication yes/' /etc/ssh/sshd_config && \
service ssh start && \
ss -tlnp | grep 22
"
```



------

# 📌 配置文件参考

## `~/.ssh/config`

```bash
Host p4-dev
  HostName localhost
  User root
  Port 2222
  PasswordAuthentication yes
  
```

------

## 容器创建命令

```bash
docker run -d \
  -v $(pwd):/workspace \
  -w /workspace \
  --privileged \
  --name p4-dev \
  -p 2222:22 \
  p4-ubuntu-full:v2.0 \
  tail -f /dev/null
```

------

# ✅ 验证清单

- □ 

  容器内 SSH 已安装（`which sshd`）

- □ 

  root 密码已设置（`echo "root:123456" | chpasswd`）

- □ 

  SSH 配置已修改（`PermitRootLogin yes`）

- □ 

  SSH 服务已启动（`service ssh start`）

- □ 

  宿主机 2222 端口监听（`ss -tlnp | grep 2222`）

- □ 

  VS Code 配置已添加（`~/.ssh/config`）

- □ 

  VS Code 能连接并输入密码

------

完成这些步骤后，你就可以在 VS Code 里像操作本地文件一样远程编辑容器里的代码了！🎉

