```bash
# 查看wsl允许安装的列表
wsl --list --online
    
# 
wsl --list -v

wsl --set-default [子系统名字]

wsl -d ubuntu

#卸载子系统
wsl --unregister [子系统名字]
```





```bash
wsl --install -d Ubuntu-20.04

wsl --export Ubuntu-20.04 D:\Desktop\Ubuntu-20.04.tar


```





```bash
git config --global --unset http.proxy 
git config --global --unset https.proxy
git config --global http.proxy http://127.0.0.1:7897
```

