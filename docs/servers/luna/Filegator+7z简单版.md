> ### Filegator+7z 简单实现内外网文件的加密流转(恢复记忆版)

---

## Filegator的容器部署以及优化

### 一、安装

~~ bash
docker run -d \
  --name filegator \
  -p 8888:80 \
  -e APACHE_PORT=80 \
  --restart unless-stopped \
  -v /home/yuuki/filegator_files:/var/www/filegator/repository \
  filegator/filegator
~~

默认账户:admin

默认密码:admin123

### 二、 优化 

执行：

```shell
docker inspect filegator --format '{{range .Mounts}}{{println .Name}}{{end}}'
```

应该会得到类似：

```shell
39a81e240213951e73d369ed53c15c7bdbef9eeb487121bdf964f4d87f38a3c3
```

### 第一步：先把现有 private 数据复制到命名 volume

里面有 FileGator 的配置/账号

先创建新 volume：

```
docker volume create filegator_private
```

然后复制旧 volume → 新 volume：

```bash
docker run --rm \
  -v 39a81e240213951e73d369ed53c15c7bdbef9eeb487121bdf964f4d87f38a3c3:/from:ro \
  -v filegator_private:/to \
  alpine \
  sh -c 'cp -a /from/. /to/'
```

这里：

```
旧 volume
    ↓
/from

新 volume
    ↓
/to
```

而且旧 volume 是：

```
:ro
```

只读挂载，避免复制过程意外修改旧数据。

### 第二步：确认复制成功

执行：

```bash
docker run --rm \
  -v filegator_private:/data:ro \
  alpine \
  find /data -maxdepth 2 -type f -print
```

应该能看到一些 FileGator 的配置文件。

**如果能看到文件，就继续。**

### 第三步：停止旧容器

```
docker stop filegator
```

确认：

```
docker ps -a --filter name=filegator
```

应该是：

```
Exited
```

### 第四步：删除旧容器

注意：

```
docker rm filegator
```

**不会删除：**

```
/home/yuuki/filegator_files
```

也不会删除我们刚刚创建的：

```
filegator_private
```

所以可以放心。

### 第五步：创建安全版 FileGator

然后：

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

这里相比原来的容器增加了：

```
--cap-drop=ALL
```

和：

```
--security-opt=no-new-privileges:true
```

同时：

```
匿名 volume
       ↓
filegator_private
```

变成了明确命名的 volume。

### 第六步：马上验证

先：

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

```bash
User: www-data
Privileged: false
Network: bridge
ReadonlyRootfs: false
SecurityOpt: ["no-new-privileges:true"]
CapDrop: ["ALL"]
CapAdd: null
```

然后：

```bash
docker inspect filegator --format '{{range .Mounts}}{{println .Type ":" .Name ":" .Source "->" .Destination}}{{end}}'
```

应该类似：

```bash
bind : :/home/yuuki/filegator_files -> /var/www/filegator/repository
volume : filegator_private:/var/lib/docker/volumes/... -> /var/www/filegator/private
```



结果图示:

                Luna 宿主机
                    │
             Docker bridge
                    │
          ┌─────────▼─────────┐
          │     FileGator     │
          │                   │
          │  www-data         │
          │  非 privileged    │
          │  CapDrop ALL      │
          │  no-new-privileges│
          └─────────┬─────────┘
                    │
             只允许接触这些
              ┌─────┴─────┐
              ▼           ▼
       repository       private
              │           │
              ▼           ▼
     /home/yuuki/        filegator_private
     filegator_files          





                    Internet
                       │
                       │
                      FRPS
                       │
                       │
                      FRPC
                       │
                       ▼
                ┌───────────────┐
                │   FileGator   │
                │               │
                │ www-data      │ ← 非 root
                │               │
                │ bridge        │ ← 网络隔离
                │               │
                │ cap-drop ALL  │ ← 没有 capabilities
                │               │
                │ no-new-privs  │ ← 禁止提权
                │               │
                │ privileged=F  │ ← 非特权容器
                └───────┬───────┘
                        │
              ┌─────────┴─────────┐
              │                   │
              ▼                   ▼
      filegator_private    filegator_files
                              │
                              │
                              ▼
                     /home/yuuki/
                       filegator_files

---

## 7z压缩加密

### 一、重写7z压缩脚本

~~ bash
vim /usr/local/bin/myenc
~~

~~ bash
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
~~

~~ bash
chmod +x /usr/local/bin/myenc
~~



### 二、注意事项

- 版本号对-p的影响,旧7z不要用于解压,只用于加密
- 密码别忘




