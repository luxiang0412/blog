---
title: window10启用ssh-agent
date: 2020-12-10 17:22:40
tags:
---

Window 下管理ssh密钥进行无密码登录服务器，还有git使用ssh认证，
一下`PS>`指的是在Powershell环境中执行

# 启用OpenSSH客户端

```powershell
#启动ssh-agent服务并开机自启
PS> Get-Service ssh-agent | Set-Service -StartupType Automatic

#导入私钥
PS> ssh-add ~/.ssh/luxiang_rsa

#查看导入的私钥
PS> ssh-add -l
```

# 链接到Git

Window下的git bash中ssh-agent和window系统中的ssh-agent不是同一个。git不知道如何与window中的ssh-agent通信  
所有需要将window的OpenSSH路径通知git
```
PS> [Environment]::SetEnvironmentVariable("GIT_SSH", "$((Get-Command ssh).Source)", [System.EnvironmentVariableTarget]::User)
```
# 参考

- [openssh_keymanagement](https://docs.microsoft.com/en-us/windows-server/administration/openssh/openssh_keymanagement)