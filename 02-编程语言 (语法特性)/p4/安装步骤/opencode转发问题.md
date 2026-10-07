### 问题一：官方安装脚本无法获取版本信息

**现象**：
运行 `curl -fsSL https://opencode.ai/install | bash` 时，报错 `Failed to fetch version information`。

**根本原因**：
容器网络（或代理）无法正常访问 GitHub API（`api.github.com`），导致安装脚本在获取最新版本号时失败。

**解决方案**：
跳过自动获取版本，直接指定一个已知版本号进行安装：

```bash
curl -fsSL https://opencode.ai/install | bash -s -- --version 1.0.180
```

或者直接下载二进制文件手动安装。

------

### 问题二：指定端口启动时出现 `[object Object]` 错误

**现象**：
运行 `opencode --port 42572` 时，终端只输出 `[object Object]`，服务无法正常启动。

**根本原因**：
容器中设置了代理环境变量（`http_proxy`/`https_proxy`），导致 OpenCode 前端进程尝试连接本地后端服务（`http://127.0.0.1:42572`）时，请求被错误地转发给了代理服务器。代理无法处理 `127.0.0.1` 这个本地地址，连接失败，程序抛出异常。

**解决方案**：
设置 `no_proxy` 环境变量，让本地地址不走代理：

```bash
export no_proxy="127.0.0.1,localhost"
```

然后正常启动即可。为了让这个配置永久生效，可以把它写入 `~/.bashrc`。

------

### 最终工作命令

安装成功后，在容器中启动 OpenCode 的稳定命令是：

```bash
export no_proxy="127.0.0.1,localhost"
opencode --port 42572   # 端口号可自定义
```



或者简化为（如果已写入 `~/.bashrc`）：

```bash
opencode --port 42572
```

------

这两个问题的本质都是**网络配置**（外网访问 GitHub 和本地代理干扰），解决后就一切正常了。如果后续还有其他问题，随时问我～