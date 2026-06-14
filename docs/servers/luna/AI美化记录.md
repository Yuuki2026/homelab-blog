# Luna 终端美化记录
## 环境信息
- 主机名：Luna
- 系统：Debian GNU/Linux 11 (Bullseye)
- Shell：Bash 5.1
- SSH客户端：iTerm2
- 字体：支持 Nerd Font 图标

## 一、安装 Starship
### 安装命令
```bash
curl -sS https://starship.rs/install.sh | sh
```

### 版本验证
```bash
starship --version
```

## 二、启用 Starship
### 普通用户
```bash
echo 'eval "$(starship init bash)"' >> ~/.bashrc
```

### Root 用户
```bash
echo 'eval "$(starship init bash)"' >> /root/.bashrc
```

### 配置生效
```bash
source ~/.bashrc
```

## 三、配置 Starship
1. 创建配置目录
```bash
mkdir -p ~/.config
```

2. 创建配置文件
```bash
vim ~/.config/starship.toml
```

3. 应用 Nerd Font 预设
```bash
starship preset nerd-font-symbols -o ~/.config/starship.toml
```

4. 重新加载配置
```bash
source ~/.bashrc
```

## 四、修复 APT 代理问题
### 问题现象
无法连接 `192.168.1.8:7890`

### 排查代理变量
```bash
env | grep -i proxy
```
查询结果：
```
http_proxy=http://clash:密码@192.168.1.8:7890
https_proxy=http://clash:密码@192.168.1.8:7890
```

### 临时取消代理
```bash
unset http_proxy
unset https_proxy
unset HTTP_PROXY
unset HTTPS_PROXY
unset ALL_PROXY
```

### 验证修复结果
```bash
apt update
```

## 五、安装终端增强工具
```bash
apt install -y \
bat \
ncdu \
ripgrep \
fd-find \
exa \
neofetch
```

## 六、常用工具说明
### bat（增强版 cat）
查看文件：
```bash
batcat docker-compose.yml
```
建议别名：
```bash
alias cat='batcat'
```

### ripgrep (rg)（增强版 grep）
```bash
# 搜索关键词 proxy
rg proxy
# 全局搜索关键词 clash
rg clash
```

### fd（增强版 find）
```bash
# 搜索文件
fdfind session
```
建议别名：
```bash
alias fd='fdfind'
```

### exa（增强版 ls）
```bash
# 详细列表查看
exa -lah
# 树形展示目录
exa --tree
# 限制树形展示深度为2级
exa --tree -L 2
```
建议别名：
```bash
alias ls='exa --icons'
alias ll='exa -lah --icons'
alias la='exa -a --icons'
alias tree='exa --tree'
```

### ncdu（磁盘占用查看工具）
```bash
# 扫描根目录
ncdu /
# 扫描/home目录
ncdu /home
```
使用场景：
- 查找大目录
- 排查日志占用
- 统计 Docker 存储占用

### neofetch（系统信息展示）
```bash
neofetch
```
输出示例：
```
OS: Debian GNU/Linux 11
Kernel: 5.10
CPU: AMD Athlon II X3 450
Memory: 766MiB / 3666MiB
```

## 七、Nerd Font 验证
执行图标测试命令：
```bash
echo "󰣇 󰉋 󰘬 󰊢"
```
验证结果：图标正常显示，iTerm2 与 Starship 字体图标支持正常。

## 八、最终 Prompt 效果
展示示例：
```
root in  Luna in /home/yuuki
❯
```
```
yuuki @Luna ~
❯
```
支持功能：
- 用户、主机名展示
- Git 分支标识
- SSH 连接图标
- 命令执行耗时统计
- Nerd Font 图标渲染

## 九、美化Login banner
### 1.样式设计
> vim  ~/.banner
``` bash
#!/bin/bash

clear

echo "Debian 11 • Docker • 🎖️  LianPo by Mq  ⚔️  ⚔️  ⚔️ "
echo

neofetch
```

### 2.利用bashrc呈现
> vim ~/.bashrc 
> 增加内容
``` bash
#-- 添加banner
if [[ $SHLVL -eq 1 ]]; then
    ~/.banner
fi
```

## 十、当前成果
### 已完成配置项
- Starship 安装与配置、Bash 终端美化
- Nerd Font 图标适配、SSH 命令行美化
- Git 分支提示功能启用
- APT 代理异常问题修复
- 全套终端增强工具部署
- 登录自定义 Banner 设计

### 最终运行环境
Debian 11 + Bash + Starship + Nerd Font + iTerm2
配套工具：exa、bat、fd、ripgrep、ncdu、neofetch
已搭建完成现代化 Linux 运维终端环境。

