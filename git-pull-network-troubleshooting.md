# Git Pull/Push 失败排查与 SSH over 443 解决记录

## 1. 问题背景

虚拟机运行在公司网络环境中，宿主机可以正常联网。此前 GitHub 的 `git pull` 和 `git push` 均能正常使用，但在公司网络策略调整后，Git 拉取失败。

当时仓库使用的是 GitHub SSH 远程地址：

```text
git@github.com:yxazxw/rememberCard.git
```

## 2. 初步检查

### 2.1 查看仓库状态和远程地址

```bash
git status --short --branch
git remote -v
git config --get-regexp '^(remote\\.|branch\\.)'
```

检查结果显示：

- 当前分支为 `main`
- 远程仓库为 `origin`
- `origin` 使用 SSH 地址
- 本地分支比远程分支领先 1 个提交
- 工作区没有未提交修改

```text
main...origin/main [ahead 1]
```

这说明本地提交没有丢失，问题发生在访问远程仓库的过程中。

### 2.2 测试 GitHub SSH 连接

```bash
git ls-remote origin HEAD
```

失败信息：

```text
Connection reset by 20.205.243.166 port 22
fatal: Could not read from remote repository.
```

使用详细日志进一步确认：

```bash
GIT_SSH_COMMAND='ssh -vvv -o ConnectTimeout=10' git ls-remote origin HEAD
```

连接可以完成部分 SSH 密钥交换，但在端口 22 上被重置。

### 2.3 测试 HTTPS 网络访问

```bash
curl -I --max-time 15 https://github.com
git ls-remote https://github.com/yxazxw/rememberCard.git HEAD
```

HTTPS 访问成功，并能正常返回远程仓库的 HEAD：

```text
03979057d1eb008f428fc87ccfe544aa94632eeb    HEAD
```

由此可以排除以下问题：

- GitHub 仓库不存在
- 当前仓库权限失效
- DNS 完全不可用
- Git 本地仓库损坏

## 3. 根因判断

虚拟机中配置了宿主机代理：

```text
HTTP_PROXY=http://192.168.137.1:7897/
HTTPS_PROXY=http://192.168.137.1:7897/
ALL_PROXY=socks://192.168.137.1:7897/
```

但是 Git 使用 SSH 远程地址时，不会自动通过 `HTTP_PROXY` 代理 SSH 连接。SSH 默认连接 GitHub 的 TCP 22 端口。

结合以下现象：

- 昨天仍然可以正常使用
- 网络策略调整后开始失败
- 错误明确指向 `port 22`
- HTTPS 访问 GitHub 正常
- SSH 连接在握手阶段被 `Connection reset`

最终判断为：

> 公司网络策略或代理链路限制了 GitHub SSH 的 22 端口，导致连接被重置；并非 GitHub 账号权限或仓库配置问题。

## 4. 解决方案：SSH 改走 443 端口

GitHub 提供了 SSH over HTTPS 的入口：

```text
ssh.github.com:443
```

### 4.1 测试 SSH 443 端口

```bash
ssh -T -p 443 git@ssh.github.com
```

首次连接会提示确认主机指纹。确认指纹为 GitHub 官方指纹后输入：

```text
yes
```

本次确认的 ED25519 指纹为：

```text
SHA256:+DiY3wvvV6TuJJhbpZisF/zLDA0zPMSvHdkr4UvCOqU
```

认证成功时会看到类似提示：

```text
Hi yxazxw! You've successfully authenticated, but GitHub does not provide shell access.
```

该提示表示 SSH 密钥认证成功。GitHub 不提供交互式 Shell 是正常行为。

### 4.2 配置 SSH 使用 443 端口

编辑 SSH 配置文件：

```bash
mkdir -p ~/.ssh
chmod 700 ~/.ssh
vi ~/.ssh/config
```

加入以下配置：

```sshconfig
Host github.com
    HostName ssh.github.com
    User git
    Port 443
    IdentityFile ~/.ssh/id_ed25519
```

如果实际使用的私钥文件不是 `~/.ssh/id_ed25519`，应先查看：

```bash
ls -la ~/.ssh
```

并将 `IdentityFile` 修改为实际私钥路径。

### 4.3 验证配置

```bash
ssh -T git@github.com
git ls-remote origin HEAD
```

验证成功后，原有的 GitHub SSH 远程地址无需修改，因为 `~/.ssh/config` 会把：

```text
github.com:22
```

透明地转发为：

```text
ssh.github.com:443
```

## 5. 一次 HTTPS 认证提示的说明

排查过程中曾将远程地址临时改为 HTTPS：

```bash
git remote set-url origin https://github.com/yxazxw/rememberCard.git
```

此时执行推送出现：

```text
Username for 'https://github.com':
Password for 'https://yxazxw@github.com':
```

这是因为 HTTPS 方式需要使用 GitHub Token 认证。GitHub 已不再支持使用账号登录密码进行 Git over HTTPS 操作。

如果坚持使用 HTTPS：

- `Username`：填写 GitHub 用户名
- `Password`：填写 GitHub Personal Access Token（PAT）
- 不要填写 GitHub 登录密码

但本次问题已经通过 SSH over 443 解决，因此最终应将远程地址恢复为 SSH：

```bash
git remote set-url origin git@github.com:yxazxw/rememberCard.git
```

确认远程地址：

```bash
git remote -v
```

预期结果：

```text
origin  git@github.com:yxazxw/rememberCard.git (fetch)
origin  git@github.com:yxazxw/rememberCard.git (push)
```

## 6. 最终验证

```bash
git pull --rebase
git status
git push origin main
```

本次验证结果：

- `ssh -T git@github.com` 认证成功
- `git ls-remote origin HEAD` 成功
- `git pull --rebase` 成功，提示当前分支已是最新
- `git push origin main` 改回 SSH 后成功
- 不再要求输入 GitHub 用户名和密码

## 7. 后续排查速查表

| 现象 | 可能原因 | 建议操作 |
|---|---|---|
| `Connection reset ... port 22` | 网络策略限制 SSH 22 端口 | 配置 SSH over 443 |
| `Permission denied (publickey)` | SSH 密钥未加载或未授权 | 检查 `ssh-agent`、公钥和 GitHub 设置 |
| HTTPS 要求输入 Password | 使用 HTTPS 远程地址 | 使用 PAT，或改回 SSH |
| `Repository not found` | 仓库地址错误或账号无权限 | 检查远程 URL 和 GitHub 权限 |
| `Could not resolve host` | DNS 或代理问题 | 检查 DNS、代理和虚拟机网络 |

## 8. 推荐的最终配置

远程仓库保持 SSH 地址：

```text
git@github.com:yxazxw/rememberCard.git
```

SSH 配置使用 443 端口：

```sshconfig
Host github.com
    HostName ssh.github.com
    User git
    Port 443
    IdentityFile ~/.ssh/id_ed25519
```

日常使用：

```bash
git pull --rebase
git push origin main
```

这样既保留了 SSH 密钥认证的便利性，也避开了公司网络对 TCP 22 端口的限制。
