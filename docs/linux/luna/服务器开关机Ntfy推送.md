# 服务器开机/关机 Ntfy 推送提醒配置文档

## 文档说明

- 功能：服务器**开机完成、关机/重启**时，自动通过 Ntfy 推送通知到安卓手机

- 适配环境：Ubuntu/Debian 系 Linux

- 核心优势：规避系统代理、网络超时容错、不影响开关机速度、全程后台自启

- 编辑器：全程使用 **vim**（替代 nano）

- 专属 Ntfy 主题：`xxxxxxxxx(替换为自己的)`

## 前置准备

1. 安卓手机安装 Ntfy 客户端，订阅主题：`xxxxxxxx(输入自己的)`

2. 服务器已安装 curl：`sudo apt install curl -y`

3. 服务器已清理无效代理配置，脚本内置清空代理逻辑，杜绝连接超时

# 一、配置服务器开机推送提醒

### 1\. 编写开机通知脚本

创建开机推送脚本文件：

```bash
sudo vim /usr/local/bin/boot-notify.sh
```

粘贴以下完整脚本内容（已优化网络等待、清空代理、信息展示）：

```bash
#!/bin/bash

# Ntfy 专属主题
TOPIC="(输入自己的主题)"
NTFY_URL="https://ntfy.sh/$TOPIC"

# 等待系统网络完全初始化（解决开机无网推送失败问题）
sleep 20

# 推送消息内容（展示主机名、IP、开机时间）
MSG="✅ 服务器开机完成
主机名: $(hostname)
IP: $(hostname -I)
时间: $(date '+%Y-%m-%d %H:%M:%S')"

# 强制清空全局代理，避免推送走代理超时
unset http_proxy https_proxy all_proxy

# 发送推送（3秒超时兜底，不阻塞系统）
curl -m 3 \
     -H "Title: 服务器开机提醒" \
     -H "Priority: high" \
     -d "$MSG" \
     "$NTFY_URL"
```

**vim 保存退出操作**：按 `Esc` → 输入 `:wq` → 回车

### 2\. 添加脚本执行权限

```bash
sudo chmod +x /usr/local/bin/boot-notify.sh
```

### 3\. 创建 systemd 开机服务

创建系统服务配置文件：

```bash
sudo vim /etc/systemd/system/boot-notify.service
```

粘贴以下服务配置：

```ini
[Unit]
Description=服务器开机 Ntfy 推送通知服务
After=network-online.target

[Service]
Type=oneshot
ExecStart=/usr/local/bin/boot-notify.sh
User=yuuki

[Install]
WantedBy=multi-user.target
```

保存退出（`Esc` → `:wq` → 回车）

### 4\. 启用并测试开机服务

```bash
# 重载系统服务配置
sudo systemctl daemon-reload

# 设置开机自启
sudo systemctl enable boot-notify

# 手动测试推送（无需重启服务器）
sudo systemctl start boot-notify

# 查看服务运行日志（排查报错）
journalctl -u boot-notify
```

执行测试命令后，手机收到推送即代表配置成功。

# 二、配置服务器关机/重启推送提醒

### 1\. 编写关机通知脚本

创建关机推送脚本文件：

```bash
sudo vim /usr/local/bin/shutdown-notify.sh
```

粘贴以下完整脚本内容：

```bash
#!/bin/bash

# Ntfy 专属主题
TOPIC="(输入自己的主题)"
NTFY_URL="https://ntfy.sh/$TOPIC"

# 推送消息内容
MSG="⚠️ 服务器即将关机/重启
主机名: $(hostname)
时间: $(date '+%Y-%m-%d %H:%M:%S')"

# 强制清空全局代理
unset http_proxy https_proxy all_proxy

# 快速推送（3秒超时，不阻塞关机流程）
curl -m 3 \
     -H "Title: 服务器关机提醒" \
     -H "Priority: high" \
     -d "$MSG" \
     "$NTFY_URL"
```

保存退出（`Esc` → `:wq` → 回车）

### 2\. 添加脚本执行权限

```bash
sudo chmod +x /usr/local/bin/shutdown-notify.sh
```

### 3\. 创建 systemd 关机服务

创建关机专属服务配置：

```bash
sudo vim /etc/systemd/system/shutdown-notify.service
```

粘贴以下配置（核心：不阻塞关机流程、优先执行推送）：

```ini
[Unit]
Description=服务器关机/重启 Ntfy 推送通知服务
DefaultDependencies=no
Before=shutdown.target reboot.target halt.target

[Service]
Type=oneshot
ExecStart=/usr/local/bin/shutdown-notify.sh
TimeoutSec=10

[Install]
WantedBy=shutdown.target reboot.target halt.target
```

保存退出

### 4\. 启用关机推送服务

```bash
# 重载服务配置
sudo systemctl daemon-reload

# 绑定关机/重启事件自触发
sudo systemctl enable shutdown-notify

# 手动测试脚本
sudo /usr/local/bin/shutdown-notify.sh

# 查看运行日志
journalctl -u shutdown-notify
```

# 三、关键优化说明（必看）

- **不影响开关机速度**：关机服务采用并行执行模式，curl 设置3秒超时、服务10秒兜底超时，不会卡死系统

- **代理彻底规避**：所有脚本内置 `unset` 清空代理命令，完美解决之前 Clash 代理超时问题

- **开机容错**：开机脚本延迟20秒执行，等待网卡、网络服务完全初始化，杜绝开机无网推送失败

- **优先级适配**：推送设置高优先级，手机弹窗提醒，和 Bark 强提醒效果一致

# 四、常用运维命令

```bash
# 关闭开机推送自启
sudo systemctl disable boot-notify

# 关闭关机推送自启
sudo systemctl disable shutdown-notify

# 实时查看开机推送日志
journalctl -fu boot-notify

# 实时查看关机推送日志
journalctl -fu shutdown-notify
```

# 五、最终效果

1. 服务器开机完成后20秒，手机收到【服务器开机提醒】推送，包含主机信息、开机时间

2. 服务器执行关机/重启瞬间，手机收到【服务器关机提醒】推送

3. 全程无卡顿、无报错，稳定复用个人专属 Ntfy 通道

> （注：文档部分内容可能由 AI 生成）

