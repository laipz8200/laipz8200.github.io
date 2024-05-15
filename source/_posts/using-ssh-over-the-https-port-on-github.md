---
title: 通过 HTTPS 的端口用 SSH 访问 GitHub
category: programming
tags:
  - ssh
  - github
  - ssl
  - tips
date: 2024-04-14 22:53:38
---

> 当我们使用代理时，可能会遇到无法通过 `ssh` 方式访问 GitHub 的情况。本文旨在分享一个简易的解决方法。️

<!-- more -->

---

## 通过SSL连接的测试

我们了解 `ssh` 默认使用端口22进行连接，但GitHub允许我们通过443端口访问。首先，我们需要验证这个方法是否可行。

在终端中输入以下命令：

```shell
ssh -T -p 443 git@ssh.github.com
```

如果成功，你将看到如下输出：

```
Hi USERNAME! You've successfully authenticated, but GitHub does not provide shell access.
```

这表明你可以使用此方法。现在，你可以通过以下命令克隆仓库：

```shell
git clone ssh://git@ssh.github.com:443/YOUR-USERNAME/YOUR-REPOSITORY.git
```

## SSH配置的修改

频繁手动更改地址显然不便，也会影响第三方工具的兼容性。好在我们可以通过调整`ssh`配置来解决。

只需在 `~/.ssh/config` 文件中添加以下内容：

```text
Host github.com
    Hostname ssh.github.com
    Port 443
    User git
```

## 再次进行测试

现在，无需指定端口即可进行测试：

```shell
$ ssh -T git@github.com
```

输出应为：

```
Hi USERNAME! You've successfully authenticated, but GitHub does not provide shell access.
```

祝你使用愉快！️

## References

[Using SSH over the HTTPS port - GitHub Docs](https://docs.github.com/en/authentication/troubleshooting-ssh/using-ssh-over-the-https-port)
