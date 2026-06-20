# 唤醒配置成命令
![qihoo370T7](https://encrypted-tbn0.gstatic.com/images?q=tbn:ANd9GcTpaU6grt81avxg5qyYGlt7quswfElEdMThOzPW4BxCAQ&s=10)
## 为什么会有这篇简单的文章
  每次都要输入etherwake -i br-lan 00:E0:66:CA:C5:E9唤醒,导致我想像linux中一样,做个别名写入bashrc中.然后我去搜索op系统中的路径,完全忘记了还有写命令这一个方法(用习惯alias了)
  其实用别名有点弊端

  - cron 不会读取 alias
  - 不可被脚本调用
  - 依赖登陆

## op中相当于bashrc的目录文件在哪里?
### shell 初始化文件（shell initialization files）或者说 启动配置文件（startup / environment configuration files）在哪里?
``` bash
/etc/profile #全局登录 shell 初始化脚本（global login shell init）

/etc/profile.d/*.sh # 模块化 shell 初始化脚本（modular init scripts）
```

## 配置命令的好处
### 1、兼容自动化
可以直接：
``` bash
echo "0 8 * * * wol-pc" >> /etc/crontabs/root
```
不会有任何问题。

### 2、灵活性高---可以扩展逻辑
例如:
``` bash
#!/bin/sh

echo "Waking PC..."
etherwake -i br-lan 00:E0:66:CA:C5:E9

logger "PC wake triggered"
```
### 3、灵活性高--可参数化
``` bash 
vim /usr/bin/wol 
```
``` bash
#!/bin/sh
#
# =============================================================================
#  WOL (Wake-on-LAN) Utility Script for OpenWrt
# =============================================================================
#
#  Description:
#    Unified Wake-on-LAN tool for multiple devices.
#    Supports parameterized device selection.
#
#  Usage:
#    wol <device>
#
#  Examples:
#    wol luna
#    wol lianpo
#
#  Add devices:
#    Edit DEVICE section in this script
#
#  Requirements:
#    etherwake installed
#    LAN interface: br-lan
#
# =============================================================================

IFACE="br-lan"

# -----------------------------------------------------------------------------
# Device Definitions (MAC Address Map)
# -----------------------------------------------------------------------------
luna_mac="00:E0:66:CA:C5:E9"
lianpo_mac="00:E0:70:2E:D2:06"

# -----------------------------------------------------------------------------
# Function: wake device
# -----------------------------------------------------------------------------
wake() {
    name="$1"
    mac="$2"

    if [ -z "$mac" ]; then
        echo "[ERROR] Unknown device: $name"
        exit 1
    fi

    echo "[INFO] Waking device: $name ($mac)"
    etherwake -i "$IFACE" "$mac"
}

# -----------------------------------------------------------------------------
# Main Logic
# -----------------------------------------------------------------------------
case "$1" in
    luna)
        wake "luna" "$luna_mac"
        ;;
    lianpo)
        wake "lianpo" "$lianpo_mac"
        ;;
    all)
        wake "luna" "$luna_mac"
        wake "lianpo" "$lianpo_mac"
        ;;
    *)
        echo "Usage: wol {luna|lianpo|all}"
        exit 1
        ;;
esac
```
``` bash 
chmod +x /usr/bin/wol
```

> 使用方法  
> wol luna   
> wol lianpo  

后续如有添加,加一行case就行
``` bash 
server3_mac="xx:xx"
```
**灵活性很高,可以加入ping命令,判断掉线的设备唤醒**

