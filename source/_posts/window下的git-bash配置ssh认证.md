---
title: window下的git bash配置ssh认证
date: 2020-12-10 17:33:50
tags:
---

# 配置

git使用密码不方便。下面是window下git bash使用ssh认证

```bash
#生成公钥 -t 算法类型 -b 长度 -C 邮箱
ssh-keygen -t rsa -b 4096 -C "email" -f ~/.ssh/luxiang_rsa

# Forces generation of Bourne shell (/bin/sh) commands on stdout. By default the shell is automatically detected.
eval $(ssh-agent -s)

# 添加私钥路径 填写你自己的路径 默认是~/.ssh/id_rsa
ssh-add ~/.ssh/luxiang_rsa

# 查看添加密钥
ssh-add -l

# 清除密钥
ssh-add -D
```

上述操作只会在当前终端中生效，要使每次启动git bash都有效，执行一下操作


将一下内容写入到`~/bashrc`中去，注意私钥路径填写你自己的。
```bash
env=~/.ssh/agent.env

agent_load_env () { test -f "$env" && . "$env" >| /dev/null ; }

agent_start () {
    (umask 077; ssh-agent >| "$env")
    . "$env" >| /dev/null ; }

agent_load_env

# agent_run_state: 0=agent running w/ key; 1=agent w/o key; 2= agent not running
agent_run_state=$(ssh-add -l >| /dev/null 2>&1; echo $?)

if [ ! "$SSH_AUTH_SOCK" ] || [ $agent_run_state = 2 ]; then
    agent_start
    ssh-add ~/.ssh/luxiang_rsa
elif [ "$SSH_AUTH_SOCK" ] && [ $agent_run_state = 1 ]; then
    ssh-add ~/.ssh/luxiang_rsa
fi

unset env
```

# 参考

- [auto_ssh](https://docs.github.com/en/free-pro-team@latest/github/authenticating-to-github/working-with-ssh-key-passphrases#auto-launching-ssh-agent-on-git-for-windows)