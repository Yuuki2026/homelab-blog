# FileGator + 7z 简单实现内外网文件的加密流转(完整版)

------

## 一、整体思路

我的需求是让文件可以在内网与外网之间方便地流转，同时尽量降低文件泄露和服务器被攻击的风险。

整体架构如下：

```text
                    Internet
                       │
                       │
                      FRPS
                       │
                       │ FRP Tunnel
                       │
                      FRPC
                       │
                       ▼
                ┌───────────────┐
                │   FileGator   │
                │               │
                │  Docker       │
                │  www-data     │
                │  bridge       │
                │  cap-drop ALL │
                │  no-new-privs │
                └───────┬───────┘
                        │
               ┌────────┴────────┐
               │                 │
               ▼                 ▼
        repository           private
               │                 │
               ▼                 ▼
 /home/yuuki/filegator_files   filegator_private
```

文件本身使用 `7z` 加密，FileGator 负责文件的上传、下载和管理，FRP 负责内外网之间的连接。

因此这里实际上有三层不同的安全措施：

```text
7z
 ↓
保护文件内容

FileGator Docker 隔离
 ↓
降低 Web 服务被攻破后对宿主机的影响

FRP / HTTPS
 ↓
负责网络传输过程中的安全
```

需要特别注意：

> **7z 加密、Docker 隔离和 HTTPS 解决的是三个不同的问题，不能互相替代。**

------

## 二、FileGator 的容器部署以及优化

### 1. 安装

首先创建 FileGator 容器：

```bash
docker run -d \
  --name filegator \
  -p 8888:80 \
  -e APACHE_PORT=80 \
  --restart unless-stopped \
  -v /home/yuuki/filegator_files:/var/www/filegator/repository \
  filegator/filegator
```

这里：

```text
/home/yuuki/filegator_files
        ↓
/var/www/filegator/repository
```

宿主机上的 `/home/yuuki/filegator_files` 就是 FileGator 实际管理的文件目录。

> FileGator 的管理员账号和初始密码应以当前版本的官方安装说明或首次初始化结果为准，不建议在文档中固定写死默认密码。

------

## 三、FileGator 容器安全优化

原始容器能够正常工作，但还可以进一步限制容器权限。

这里主要采用：

1. 非 root 用户运行
2. 删除全部 Linux Capabilities
3. 禁止通过 setuid 等机制获得新的权限
4. 不使用 privileged 容器
5. 不挂载 Docker Socket
6. 只向容器提供必要的数据目录
7. 将 FileGator 的 `private` 数据从匿名 volume 转换为命名 volume

------

### 1. 查看原来的 private volume

执行：

```bash
docker inspect filegator --format '{{range .Mounts}}{{println .Name}}{{end}}'
```

可能得到类似：

```text
39a81e240213951e73d369ed53c15c7bdbef9eeb487121bdf964f4d87f38a3c3
```

这是 Docker 自动创建的匿名 volume。

FileGator 的 `private` 目录通常保存了账号、配置、日志等数据，因此在删除旧容器之前需要先把它保存下来。

------

### 2. 创建命名 volume

创建一个更加明确的 volume：

```bash
docker volume create filegator_private
```

以后 FileGator 的私有数据就统一保存在：

```text
filegator_private
```

中。

相比随机生成的匿名 volume，命名 volume 更方便管理和迁移。

------

### 3. 将旧 private 数据复制到新的 volume

使用一个临时 Alpine 容器进行复制：

```bash
docker run --rm \
  -v 39a81e240213951e73d369ed53c15c7bdbef9eeb487121bdf964f4d87f38a3c3:/from:ro \
  -v filegator_private:/to \
  alpine \
  sh -c 'cp -a /from/. /to/'
```

这里的关系是：

```text
旧 volume
    │
    ▼
  /from
    │
    │ cp -a
    ▼
  /to
    │
    ▼
filegator_private
```

旧 volume 使用：

```text
:ro
```

进行只读挂载。

这样复制过程中不会修改原来的数据。

------

### 4. 确认复制成功

执行：

```bash
docker run --rm \
  -v filegator_private:/data:ro \
  alpine \
  find /data -maxdepth 2 -type f -print
```

正常情况下应该可以看到类似：

```text
/data/users.json
/data/.htaccess
/data/logs/app.log
/data/logs/.gitignore
/data/users.json.blank
/data/.gitignore
/data/sessions/.gitignore
```

如果能够看到 FileGator 的数据文件，就可以继续。

------

### 5. 停止旧容器

```bash
docker stop filegator
```

然后确认：

```bash
docker ps -a --filter name=filegator
```

应该可以看到：

```text
Exited
```

------

### 6. 删除旧容器

```bash
docker rm filegator
```

这里需要区分：

```text
容器
volume
bind mount
```

`docker rm filegator` 删除的是**容器本身**。

不会删除：

```text
/home/yuuki/filegator_files
```

也不会删除：

```text
filegator_private
```

因此前面迁移到命名 volume 的目的，就是让容器可以安全地删除和重新创建，而不会丢失 FileGator 的账号和配置。

------

## 四、创建安全版 FileGator

重新创建容器：

```bash
docker run -d \
  --name filegator \
  -p 8888:80 \
  -e APACHE_PORT=80 \
  --restart unless-stopped \
  --cap-drop=ALL \
  --security-opt=no-new-privileges:true \
  -v /home/yuuki/filegator_files:/var/www/filegator/repository \
  -v filegator_private:/var/www/filegator/private \
  filegator/filegator:latest
```

相比最初的容器，主要增加了：

```text
--cap-drop=ALL
```

以及：

```text
--security-opt=no-new-privileges:true
```

同时将：

```text
匿名 volume
     ↓
filegator_private
```

变成了明确命名的 volume。

------

## 五、这些安全选项分别有什么作用？

### 1. www-data：非 root 用户

FileGator 当前使用：

```text
www-data
```

运行，而不是 root。

因此即使 FileGator 存在漏洞并被攻击者利用，攻击者首先获得的也是：

```text
www-data
```

而不是 root。

可以理解为：

```text
Web 漏洞
   ↓
攻击者进入容器
   ↓
www-data
```

而不是：

```text
Web 漏洞
   ↓
root
```

这是第一层权限限制。

------

### 2. `--cap-drop=ALL`

```bash
--cap-drop=ALL
```

Linux 的 root 权限实际上可以拆分成许多不同的 Capability。

例如：

```text
CAP_NET_ADMIN
CAP_SYS_ADMIN
CAP_NET_RAW
CAP_CHOWN
...
```

普通 Web 应用通常不需要这些额外能力。

因此直接删除：

```text
ALL Capabilities
```

可以进一步缩小 FileGator 的权限。

也就是说：

```text
www-data
    +
CapDrop ALL
```

比单纯使用非 root 用户更加严格。

------

### 3. `no-new-privileges`

```bash
--security-opt=no-new-privileges:true
```

作用是禁止进程通过某些机制获得新的、更高的权限。

例如攻击者已经获得：

```text
www-data
```

即使容器内部存在某些可以尝试提权的机制，也不能简单地通过这些机制获得额外权限。

因此：

```text
cap-drop=ALL
```

解决的是：

> **不给它额外能力。**

而：

```text
no-new-privileges
```

解决的是：

> **以后也不要通过某些机制再获得新的权限。**

两者属于不同的防线。

------

### 4. 不使用 `privileged`

容器保持：

```text
Privileged: false
```

不要使用：

```bash
--privileged
```

因为 privileged 容器拥有大量额外权限，会明显削弱 Docker 与宿主机之间的隔离。

------

### 5. 不挂载 Docker Socket

不要给 FileGator 挂载：

```text
/var/run/docker.sock
```

例如不要使用：

```bash
-v /var/run/docker.sock:/var/run/docker.sock
```

因为 Docker Socket 本身具有很高的 Docker 控制权限。

如果攻击者先通过 FileGator 漏洞进入容器，又能够访问 Docker Socket，就可能进一步尝试控制其他容器甚至宿主机。

因此：

```text
FileGator
    X
Docker Socket
```

是非常重要的一条安全边界。

------

### 6. 只挂载必要目录

FileGator 只需要访问自己的文件目录：

```text
/home/yuuki/filegator_files
```

因此不要为了方便直接挂载：

```text
/
```

或者：

```text
/home/yuuki
```

甚至：

```text
/root
```

正确思路是：

> **应用需要什么，就给它什么；不需要的东西不要给。**

这就是最小权限原则。

------

## 六、验证容器安全配置

首先：

```bash
docker ps
```

然后：

```bash
docker inspect filegator --format '
User: {{.Config.User}}
Privileged: {{.HostConfig.Privileged}}
Network: {{.HostConfig.NetworkMode}}
ReadonlyRootfs: {{.HostConfig.ReadonlyRootfs}}
SecurityOpt: {{json .HostConfig.SecurityOpt}}
CapDrop: {{json .HostConfig.CapDrop}}
CapAdd: {{json .HostConfig.CapAdd}}
'
```

理想结果：

```text
User: www-data
Privileged: false
Network: bridge
ReadonlyRootfs: false
SecurityOpt: ["no-new-privileges:true"]
CapDrop: ["ALL"]
CapAdd: null
```

其中：

```text
ReadonlyRootfs: false
```

目前是正常的。

因为暂时没有启用只读根文件系统，以避免 FileGator 因为某些运行时目录需要写权限而无法正常工作。

后续如果需要进一步加固，可以单独测试：

```text
--read-only
```

以及：

```text
--tmpfs /tmp
```

但这属于下一阶段优化。

------

## 七、验证 Volume

执行：

```bash
docker inspect filegator --format '{{range .Mounts}}{{println .Type ":" .Name ":" .Source "->" .Destination}}{{end}}'
```

应该可以看到类似：

```text
bind : :/home/yuuki/filegator_files -> /var/www/filegator/repository
volume : filegator_private:/var/lib/docker/volumes/... -> /var/www/filegator/private
```

其中最重要的是：

```text
/home/yuuki/filegator_files
        ↓
/var/www/filegator/repository
```

以及：

```text
filegator_private
        ↓
/var/www/filegator/private
```

------

## 八、最终的 FileGator 安全边界

```text
                         Luna 宿主机
                              │
                       Docker bridge
                              │
                 ┌────────────▼────────────┐
                 │        FileGator        │
                 │                         │
                 │       www-data          │
                 │       非 root           │
                 │                         │
                 │   Privileged = false    │
                 │                         │
                 │     CapDrop = ALL       │
                 │                         │
                 │ no-new-privileges=true  │
                 └────────────┬────────────┘
                              │
                       只能访问必要数据
                              │
                 ┌────────────┴────────────┐
                 │                         │
                 ▼                         ▼
             repository                 private
                 │                         │
                 ▼                         ▼
     /home/yuuki/filegator_files    filegator_private
```

因此即使未来 FileGator 出现 Web 漏洞：

```text
FileGator 漏洞
      ↓
获得 WebShell / RCE
      ↓
首先只能控制 www-data
      ↓
没有额外 Capabilities
      ↓
不能通过简单的提权机制获得新权限
      ↓
没有 privileged
      ↓
没有 Docker Socket
      ↓
只能看到被挂载的数据
```

这样可以显著降低：

> **FileGator 被攻破 → Luna 宿主机被直接攻破**

的风险。

------

## 九、7z 压缩加密

FileGator 解决的是文件的上传、下载和管理，而真正需要长期保存或通过 FileGator 流转的敏感文件，则使用 `7z` 进行加密。

这样即使 FileGator 的文件目录被直接读取，攻击者看到的也只是：

```text
backup.7z
```

而不是里面的明文文件。

------

### 1. 安装 7z

Debian 系统可以安装：

```bash
apt install p7zip-full
```

检查：

```bash
7z
```

或者：

```bash
7z i
```

当前使用的版本可以通过：

```bash
7z
```

查看。

------

### 2. 7z 加密参数

这里使用：

```bash
7z a \
  -t7z \
  -mx=9 \
  -mhe=on \
  -p \
  archive.7z \
  file
```

各参数含义：

```text
-t7z
```

使用 7z 格式。

```text
-mx=9
```

使用最高压缩等级。

```text
-mhe=on
```

加密文件头，也就是同时隐藏压缩包中的文件名。

```text
-p
```

交互式输入密码。

------

## 十、将 7z 加密封装成脚本

为了避免每次都输入完整命令，可以创建：

```text
/usr/local/bin/myenc
```

内容：

```bash
#!/usr/bin/env bash

set -e

if [ "$#" -lt 2 ]; then
    echo "用法: myenc <压缩包名.7z> <文件/文件夹>..."
    exit 1
fi

archive="$1"
shift

7z a \
    -t7z \
    -mx=9 \
    -mhe=on \
    -p \
    "$archive" \
    "$@"
```

赋予执行权限：

```bash
chmod +x /usr/local/bin/myenc
```

以后就可以直接：

```bash
myenc test.7z file.txt
```

或者：

```bash
myenc backup.7z dir/
```

执行过程中会交互式要求输入密码。

------

## 十一、7z 加密的注意事项

### 1. 密码不能忘

7z 加密没有方便的“找回密码”机制。

如果密码丢失：

```text
文件还在
+
7z 压缩包还在
+
没有密码
=
基本无法恢复文件内容
```

因此重要文件的密码应该进行可靠的离线保存。

------

### 2. 使用 `-mhe=on`

建议始终使用：

```text
-mhe=on
```

否则即使文件内容被加密，压缩包中的文件名可能仍然可以被看到。

例如没有文件头加密：

```text
secret.7z
├── passport.jpg
├── bank.xlsx
└── password.txt
```

而启用：

```text
-mhe=on
```

之后，文件名也会被加密。

------

### 3. 不要把密码写进命令行

不要这样：

```bash
7z a -p12345678 archive.7z file
```

因为命令行参数可能出现在：

```text
shell history
进程信息
日志
```

中。

推荐使用：

```bash
-p
```

让 7z 交互式询问密码。

------

### 4. 压缩包本身不等于安全

7z 只能保护：

```text
文件内容
```

它不能保护：

```text
FileGator 登录密码
FileGator Session
HTTP 请求
HTTP Cookie
文件上传过程
文件下载过程
```

因此：

```text
7z
```

和：

```text
HTTPS
```

解决的是不同的问题。

------

## 十二、7z + FileGator + HTTPS 的完整安全模型

最终希望形成：

```text
                 文件发送方
                     │
                     │
               7z 加密 + 密码
                     │
                     ▼
              ┌─────────────┐
              │   archive   │
              │    .7z      │
              └──────┬──────┘
                     │
                     │ HTTPS
                     ▼
                   FRPS
                     │
                     │ FRP
                     ▼
                   FRPC
                     │
                     ▼
              ┌──────────────┐
              │  FileGator   │
              │              │
              │ www-data     │
              │ CapDrop ALL  │
              │ no-new-privs │
              │ bridge       │
              └──────┬───────┘
                     │
                     ▼
             /home/yuuki/
             filegator_files
```

这里每一层负责不同的问题：

| 防护措施             | 主要解决的问题             |
| -------------------- | -------------------------- |
| 7z + 密码            | 文件内容泄露               |
| `-mhe=on`            | 文件名泄露                 |
| FileGator `www-data` | 降低 Web 服务权限          |
| `--cap-drop=ALL`     | 删除不必要的 Linux 权限    |
| `no-new-privileges`  | 限制进一步提权             |
| `Privileged=false`   | 保持容器与宿主机隔离       |
| 不挂载 Docker Socket | 防止通过 Docker 控制宿主机 |
| 最小化 Volume        | 限制容器能看到的数据       |
| Docker bridge        | 提供网络隔离               |
| HTTPS                | 保护登录、Cookie、文件传输 |
| FRP                  | 提供内外网之间的连接通道   |

------

## 十三、最终的安全目标

这里并不是追求“绝对安全”，而是让攻击者必须连续突破多层防线。

理想情况下：

```text
              Web 攻击
                  │
                  ▼
            FileGator RCE
                  │
                  ▼
             www-data
                  │
          ┌───────┴───────┐
          │               │
     CapDrop ALL     no-new-privileges
          │               │
          └───────┬───────┘
                  ▼
          普通受限容器
                  │
       ┌──────────┼──────────┐
       │          │          │
       ▼          ▼          ▼
   无 privileged  无 docker   无宿主机
                  socket      根目录
                  │
                  ▼
             容器隔离
                  │
                  ▼
               Luna
```

而即使攻击者最终拿到了：

```text
/home/yuuki/filegator_files
```

里面的重要文件仍然可以是：

```text
important.7z
```

而不是明文：

```text
important.pdf
important.jpg
password.txt
```

这就是这套方案的核心：

> **Docker 隔离保护服务器，7z 加密保护文件，HTTPS 保护传输过程。**

三者结合起来，比单独依赖其中任何一个措施都更加可靠。
