# Git + GitHub + MkDocs 技术博客搭建笔记

## 项目目标

实现：

```text
本地 Markdown
    ↓
Git
    ↓
GitHub
    ↓
GitHub Actions
    ↓
GitHub Pages
    ↓
公开博客
```

最终访问地址：

```text
https://yuuki2026.github.io/homelab-blog/
```

---

# MkDocs 安装

安装：

```bash
pip install mkdocs
pip install mkdocs-material
```

创建项目：

```bash
mkdocs new homelab-blog
cd homelab-blog
```

启动本地预览：

```bash
mkdocs serve
```

访问：

```text
http://127.0.0.1:8000
```

---

# Material 主题

最小配置：

```yaml
site_name: Yuuki's Homelab

theme:
  name: material
```

推荐配置：

```yaml
site_name: Yuuki's Homelab
site_url: https://yuuki2026.github.io/homelab-blog/

theme:
  name: material

  features:
    - content.code.copy

  palette:

    - scheme: default
      toggle:
        icon: material/weather-night

    - scheme: slate
      toggle:
        icon: material/weather-sunny
```

功能：

* Material主题
* 代码复制按钮
* 深色模式
* 浅色模式

---

# Git 初始化

初始化仓库：

```bash
git init
git add .
git commit -m "Initial commit"
```

关联远程仓库：

```bash
git remote add origin git@github.com:Yuuki2026/homelab-blog.git
```

查看：

```bash
git remote -v
```

---

# SSH Key 配置

## 为什么不用 HTTPS

HTTPS：

```text
每次可能需要认证
```

SSH：

```text
公钥认证
更适合长期使用
```

---

## 创建 GitHub 专用密钥

生成：

```bash
ssh-keygen -t ed25519 \
-C "github-Yuuki2026" \
-f ~/.ssh/id_ed25519_github
```

生成：

```text
~/.ssh/id_ed25519_github
~/.ssh/id_ed25519_github.pub
```

---

## 上传公钥

查看：

```bash
cat ~/.ssh/id_ed25519_github.pub
```

复制内容。

进入：

```text
GitHub
→ Settings
→ SSH and GPG Keys
→ New SSH Key
```

添加公钥。

---

# SSH Config 配置

文件：

```text
~/.ssh/config
```

配置：

```ssh
Host github.com
    HostName ssh.github.com
    Port 443
    User git
    IdentityFile ~/.ssh/id_ed25519_github
    IdentitiesOnly yes
```

---

# 为什么使用 443 端口

默认：

```text
github.com:22
```

部分网络环境可能阻断：

```text
22/tcp
```

改为：

```text
ssh.github.com:443
```

优点：

* 穿透能力更强
* 与 HTTPS 同端口
* 更稳定

本质仍然是：

```text
SSH协议
```

并不是 HTTPS 登录。

---

# SSH 测试

测试：

```bash
ssh -T git@github.com
```

成功：

```text
Hi Yuuki2026!
You've successfully authenticated.
```

---

# SSH Agent

避免每次输入密钥密码：

启动：

```bash
eval "$(ssh-agent -s)"
```

加载：

```bash
ssh-add ~/.ssh/id_ed25519_github
```

查看：

```bash
ssh-add -l
```

之后本次开机期间无需重复输入。

---

# GitHub Pages

采用：

```text
GitHub Actions 自动部署
```

不再使用：

```text
mkdocs gh-deploy
```

---

# GitHub Actions 自动部署

目录：

```text
.github/workflows/
```

文件：

```text
mkdocs.yml
```

作用：

```text
git push
↓
自动构建
↓
自动发布
```

以后无需执行：

```bash
mkdocs gh-deploy
```

---

# Git 基础概念

Git 有三个重要区域：

```text
工作区
↓
暂存区
↓
提交历史
```

常用流程：

```bash
git add .
git commit -m "更新内容"
git push
```

---

# 分支概念

本地：

```text
main
```

远程：

```text
origin/main
```

关系：

```text
main
  ↓
origin/main
```

建立跟踪：

```bash
git push -u origin main
```

以后：

```bash
git push
git pull
```

即可。

---

# 创建分支

创建：

```bash
git checkout -b feature/openwrt
```

查看：

```bash
git branch
```

切换：

```bash
git checkout main
```

---

# 合并分支

切回主分支：

```bash
git checkout main
```

合并：

```bash
git merge feature/openwrt
```

---

# 删除分支

删除：

```bash
git branch -d feature/openwrt
```

强制删除：

```bash
git branch -D feature/openwrt
```

---

# Git Commit Hash

每个提交都有唯一 ID：

```text
34e3aa1
5124606
```

修改提交：

```bash
git commit --amend
```

会生成新的 Hash。

因此：

```text
旧提交
↓
新提交
```

Git 会认为是不同历史。

---

# Force Push

当修改历史后：

```bash
git push
```

可能出现：

```text
non-fast-forward
```

解决：

```bash
git push --force-with-lease
```

推荐使用：

```text
--force-with-lease
```

不要直接使用：

```text
--force
```

---

# 常用命令速查

查看状态：

```bash
git status
```

查看日志：

```bash
git log --oneline
```

查看分支：

```bash
git branch
```

查看远程：

```bash
git remote -v
```

拉取：

```bash
git pull
```

推送：

```bash
git push
```

查看 SSH：

```bash
ssh -T git@github.com
```

查看已加载密钥：

```bash
ssh-add -l
```

---

# 当前推荐工作流

新增文章：

```bash
vim docs/linux/docker.md
```

提交：

```bash
git add .
git commit -m "新增 Docker 笔记"
git push
```

等待 GitHub Actions 自动部署。

访问：

```text
https://yuuki2026.github.io/homelab-blog/
```

即可查看更新。

