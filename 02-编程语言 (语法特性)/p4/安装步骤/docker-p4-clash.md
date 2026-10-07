## 🎯 目的

让 **Docker 容器**通过 **宿主机 Clash 代理** 访问外网，且 Clash 只监听 `127.0.0.1`（安全），不开放局域网。

------

## 🔧 架构流程

```text
容器 (172.18.0.2)
    │
    │ 访问 http://host.docker.internal:7890
    ▼
宿主机 (172.18.0.1)
    │
    │ socat 监听 172.18.0.1:7890
    │ 转发到 127.0.0.1:7890
    ▼
Clash (127.0.0.1:7890)
    │
    │ 通过节点访问外网
    ▼
外网 (GitHub 等)
```



------

## 📋 完整操作步骤

### 1. 确保 Clash 正常运行并监听 `127.0.0.1:7890`

```bash
# 查看 Clash 状态
clashon

# 查看端口监听
ss -tlnp | grep 7890
```

**预期输出**：`127.0.0.1:7890` 被 `mihomo` 监听。

------

### 2. 启动 socat 转发（不会挂起的命令）

```bash
# 清理旧进程
sudo pkill -f socat

# 启动 socat
sudo setsid socat TCP-LISTEN:7890,fork,reuseaddr,bind=172.18.0.1 TCP:127.0.0.1:7890 < /dev/null > /dev/null 2>&1 &

# 验证
ps aux | grep socat | grep -v grep
ss -tlnp | grep 7890
```

**预期输出**：

- `socat` 进程状态为 `S`（正常）
- 端口 `172.18.0.1:7890` 和 `127.0.0.1:7890` 都在监听

------

### 3. 容器内测试

```bash
# 进入容器
docker exec -it p4-dev bash

# 测试代理
curl -x http://host.docker.internal:7890 https://github.com -I
```

**预期输出**：`HTTP/2 200`

------

## 🔧 常用命令速查

| 操作            | 命令                                                         |
| :-------------- | :----------------------------------------------------------- |
| 启动 Clash      | `clashon`                                                    |
| 停止 Clash      | `clashoff`                                                   |
| 查看 Clash 状态 | `clashctl status`                                            |
| 查看端口监听    | `ss -tlnp | grep 7890`                                       |
| 启动 socat      | `sudo setsid socat TCP-LISTEN:7890,fork,reuseaddr,bind=172.18.0.1 TCP:127.0.0.1:7890 < /dev/null > /dev/null 2>&1 &` |
| 停止 socat      | `sudo pkill -f socat`                                        |
| 查看 socat 状态 | `ps aux | grep socat | grep -v grep`                         |
| 进入容器        | `docker exec -it p4-dev bash`                                |
| 容器内测试代理  | `curl -x http://host.docker.internal:7890 https://github.com -I` |

------

## ✅ 验证清单

- □ 

  Clash 运行正常（`ss -tlnp | grep 7890` 看到 `127.0.0.1:7890`）

- □ 

  socat 运行正常（`ps aux | grep socat` 看到进程，状态 `S`）

- □ 

  端口 `172.18.0.1:7890` 在监听（`ss -tlnp | grep 7890`）

- □ 

  容器内 `curl -x http://host.docker.internal:7890 https://github.com -I` 返回 `HTTP/2 200`

------

## ⚠️ 注意事项

1. **socat 挂起问题**：用 `setsid` 启动，避免 `sudo` 等待密码导致进程挂起
2. **Clash 端口**：确认 Clash 的 `mixed-port` 是 `7890`（如果不是，修改配置或调整 socat 目标端口）
3. **容器 IP**：确认容器内 `host.docker.internal` 能解析到 `172.18.0.1`

------

现在这套流程已经完整了，可以直接用！🎉