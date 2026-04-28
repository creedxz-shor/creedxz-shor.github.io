一、备份：

层级一：纯文本记录（给人看）
这个方法是为了让你在恢复时，打开文件看一眼，知道该怎么手动分区。

操作方法： 在备份前，运行以下命令，把信息写入一个叫 disk_layout.txt 的文件：


Bash
```# 1. 记录分区结构 (fdisk -l)
echo "=== 分区表结构 ===" > disk_layout.txt
sudo fdisk -l /dev/sda >> disk_layout.txt

# 2. 记录 UUID (blkid) - 这步最重要！
echo -e "\n=== UUID 信息 (用于 fstab 和 mkfs) ===" >> disk_layout.txt
sudo blkid >> disk_layout.txt

# 3. 记录当前的挂载情况 (df -h)
echo -e "\n=== 挂载点 ===" >> disk_layout.txt
df -hT >> disk_layout.txt

# 4. 备份 fstab 文件内容
echo -e "\n=== /etc/fstab 内容 ===" >> disk_layout.txt
cat /etc/fstab >> disk_layout.txt
```

恢复时的用法： 恢复时用 cat disk_layout.txt 查看。

对着分区表，手动用 fdisk 或 gparted 建立分区。
对着 UUID 信息，修改你解压后的 /etc/fstab。

层级二：tar

你的检查非常专业！结合你提供的 findmnt、df 和 du 三份报告，局势已经完全明朗了。

结论：你的系统里“无底洞”只有一个，其他的都是真金白银的数据。
我来帮你解读这份体检报告：

确认“无底洞”：

从 findmnt 看，除了 /DATA/rclone_mount 是 fuse.rclone（网盘）外，其余所有挂载点（/mnt/disk_zhaoyang, /mnt/disk_sony）都在 /mnt 下。
对策：只要排除 /mnt 和 /DATA/rclone_mount，你就绝对安全了。
确认数据分布（du 报告分析）：

/ (根目录)：总共占用了 34GB（不含挂载点）。
/DATA：占用了 20GB。这说明你的数据库、应用配置等数据都在这里，这是实打实的本地数据，必须备份！
/var：占用了 11GB。这里主要是 Docker 的镜像缓存（overlay2）和日志。
/usr：3.9GB。系统程序。
最终行动方案
由于你的 /DATA 里有 20GB 的重要数据库，且 /var 里有 11GB 的 Docker 数据，必须严格执行“停机备份”策略。

我为你写好了一个全自动脚本。你可以直接复制粘贴到一个文件里（比如叫 backup.sh），然后用 sudo bash backup.sh 运行。它会自动处理 Docker 的启停。

Bash

```
#!/bin/bash

# 1. 定义变量
BACKUP_DIR="/mnt/disk_zhaoyang/006_software/001_os_images/nas_3865u"
DATE=$(date +%Y%m%d)
FILENAME="backup_nas_3865u_$DATE.tar.gz"

# 确保目录存在
mkdir -p "$BACKUP_DIR"

echo "=========================================="
echo "阶段 1/3: 停止 Docker 服务 (保护数据库完整性)"
echo "=========================================="
systemctl stop docker
systemctl stop docker.socket
echo "Docker 已停止。"

echo "=========================================="
echo "阶段 2/3: 开始打包系统 (预计耗时 10-20 分钟)"
echo "注意：我们将排除云盘、外接硬盘和 Docker 垃圾缓存"
echo "=========================================="

# 解释一下排除项：
# --exclude=/DATA/rclone_mount : 排除云盘
# --exclude=/var/lib/docker/overlay2 : 排除 Docker 镜像缓存(节省约5-8GB空间)，恢复后 pull 即可
# --exclude=/var/log : 排除内存日志 (Armbian 特性)

tar -cvpzf "$BACKUP_DIR/$FILENAME" \
--exclude="$BACKUP_DIR" \
--exclude=/proc --exclude=/sys --exclude=/dev --exclude=/run --exclude=/tmp \
--exclude=/mnt --exclude=/media --exclude=/lost+found \
--exclude=/DATA/rclone_mount \
--exclude=/var/lib/docker/overlay2 \
--exclude=/var/log \
/

echo "=========================================="
echo "阶段 3/3: 恢复 Docker 服务"
echo "=========================================="
systemctl start docker
echo "Docker 已恢复运行。"

echo "=========================================="
echo "备份完成！"
echo "文件位置: $BACKUP_DIR/$FILENAME"
echo "文件大小: $(ls -lh $BACKUP_DIR/$FILENAME | awk '{print $5}')"
echo "=========================================="
```

这个脚本做了什么优化？
排除 /var/lib/docker/overlay2：
你的 /var 有 11GB，其中大部分是这个。排除它后，你的备份包体积可能会从 17GB 骤降到 10GB 左右。
代价：恢复系统后，你的 Docker 容器还在，配置也在，但是镜像没了。你需要运行一次 docker compose pull 或者让它自动重新下载镜像。我觉得这个交换非常划算。
排除 /var/log：
Armbian 使用 zram 把日志挂载在内存里。备份这个目录没有意义，而且可能导致权限报错。
安全保护 /DATA：
脚本没有排除 /DATA，也没有排除 /DATA/docker_dbdata。这正是你想要的——完整的数据库备份。

### 二、还原：
第一阶段：准备虚拟机环境
下载救援 ISO： 你需要一个能启动系统的光盘镜像。推荐下载 Debian 12/13 Live ISO 或者 SystemRescue。

Debian Live 下载地址: https://www.debian.org/CD/live/ (选 amd64 -> iso-hybrid -> standard 或 gnome)

第二阶段：还原系统（核心操作）
假设你已经在虚拟机里通过 fdisk -l 看到虚拟硬盘是 /dev/sda。

分区（模拟原机结构）： 使用 cfdisk /dev/sda 或 gparted。

我们要把这块白板硬盘切成我们在前面“蓝图”里规划的三块：

BIOS Boot (为了兼容性)
EFI (为了启动)
Root (为了存数据)
推荐使用 cfdisk，它有图形化界面，比 fdisk 直观得多。

运行命令：

Bash

`cfdisk /dev/sda`

选择分区表类型： 屏幕会问你选什么 Label Type。

选中 gpt，回车。
创建分区： 你现在看到的是 Free space。

第1个分区 (BIOS Boot):

选 [ New ] -> 回车。
Size 输入：4M -> 回车。
选 [ Type ] -> 回车 -> 找到并选中 BIOS boot -> 回车。
第2个分区 (EFI):

移动光标到剩下的 Free space。
选 [ New ] -> 回车。
Size 输入：256M -> 回车。
选 [ Type ] -> 回车 -> 找到并选中 EFI System -> 回车。
第3个分区 (Root):

移动光标到剩下的 Free space。
选 [ New ] -> 回车。
Size：默认全部剩余空间 -> 回车。
选 [ Type ] -> 回车 -> 选 Linux filesystem (默认就是) -> 回车。
写入并退出:

现在你的列表里应该有 sda1, sda2, sda3。
选中底部的 [ Write ] -> 回车 -> 输入 yes 确认。
选中 [ Quit ] 退出。
第二步：格式化 (Formatting)
分好了区，现在要给它们铺上文件系统。

Bash

```
# 1. 格式化 EFI 分区 (sda2) 为 FAT32
mkfs.vfat -F32 /dev/sda2

# 2. 格式化 Root 分区 (sda3) 为 EXT4
mkfs.ext4 /dev/sda3

# 注意：sda1 是 BIOS boot，不需要格式化，留着就行。
```
第三步：挂载 (Mounting)
这是最后一步，把这些分区“插”到当前的系统目录里，这样你才能往里面写东西。

顺序很重要：先挂大房子，再挂小房间。

Bash

```
# 1. 把最大的根分区挂载到 /mnt
mount /dev/sda3 /mnt

# 2. 创建存放 EFI 分区的目录
mkdir -p /mnt/boot/efi

# 3. 把 EFI 分区挂上去
mount /dev/sda2 /mnt/boot/efi

```
下载并解压备份：

Bash

```
# 在虚拟机里输入
cd /mnt
scp root@192.168.xx.xx:/mnt/disk_zhaoyang/006_software/001_os_images/nas_3865u/backup_nas_3865u_xxxx.tar.gz .
```
(注意命令最后有个点 .，代表当前目录)

# 解压 (这一步会很久)
`tar -xvpzf backup_nas_3865u_xxxx.tar.gz -C /mnt/`

为了防止遗漏，我在下面写了一段完整的补全代码。请在虚拟机里（确保硬盘还挂在 /mnt）运行：

Bash

```
echo "开始补全目录结构..."

# 1. 系统基础目录
mkdir -p /mnt/{proc,sys,dev,run,tmp,media,mnt}
chmod 1777 /mnt/tmp

# 2. 恢复 Docker 和 日志目录结构
mkdir -p /mnt/var/lib/docker/overlay2
mkdir -p /mnt/var/log

# 3. 恢复 NAS 硬盘挂载点 (根据你的 fstab)
mkdir -p /mnt/mnt/{disk_zhaoyang,disk_sony,disk_elements,disk_t420}

# 4. 恢复 DATA 下的特殊挂载点
mkdir -p /mnt/DATA/rclone_mount

echo "目录补全完成！"

```
解压完还没结束，必须修改配置，否则开不了机。

修改 /mnt/etc/fstab：

查看新硬盘 UUID：blkid
编辑文件：nano /mnt/etc/fstab
把 / (根目录) 和 /boot/efi 的 UUID 替换成 blkid 里显示的 /dev/sda3 和 /dev/sda2 的新 UUID。

安装 GRUB 引导：

Bash

```
# 挂载虚拟系统
for i in /dev /dev/pts /proc /sys /run; do mount -B $i /mnt$i; done

# 进笼子
chroot /mnt

# 安装
grub-install /dev/sda
update-grub

# 退出
exit
```

第一步：修改 Netplan 配置文件
找到配置文件： 输入以下命令，查看文件名（通常叫 armbian-default.yaml 或 01-netcfg.yaml）：

Bash
```

ls /etc/netplan/
编辑它： 假设文件名是 armbian-default.yaml：
```

Bash

`nano /etc/netplan/armbian-default.yaml`
关键修改（请仔细看）： 你现在的配置文件里写的肯定是旧网卡名（比如 eth0 或 enp3s0）和静态 IP。 你需要把网卡名改成现在的 enp0s5，并开启 DHCP。

修改前（你可能看到的）：

YAML

```
network:
  version: 2
  renderer: NetworkManager
  ethernets:
    eth0:               <--- 这里是旧网卡，必须改！
      dhcp4: no
      addresses:
        - 192.168.1.100/24  <--- 这是你之前设的静态IP
      routes:
        - to: default
          via: 192.168.1.1
      nameservers:
        addresses: [8.8.8.8]
```
修改后（改成这样）： 注意：YAML 格式非常严格，缩进必须用空格，不能用 Tab！

YAML

```
network:
  version: 2
  renderer: NetworkManager
  ethernets:
    enp0s5:             <--- 改成现在的新网卡名
      dhcp4: true       <--- 改成 true (自动获取)
      # 下面的 addresses, routes 等全都可以删掉或注释掉
```
保存：Ctrl+O, Enter, Ctrl+X。

第二步：应用配置
修改完文件还不会生效，必须运行命令让 Netplan 读取它：

Bash

`netplan apply`

如果报错提示权限太高（permissions are too open），不用管，通常也会生效。
如果报错 YAML 格式错误，请检查缩进。

重建docker容器和镜像

由于你备份时排除了 /var/lib/docker/overlay2，目前系统中所有的容器和镜像本质上都是“损坏”的（有空壳配置，没有实际文件）。

即便你现在重新 pull 了镜像，旧的容器依然指向那些丢失的文件层，无法启动。

请执行以下“核弹级”清理命令，将 Docker 环境彻底重置为空白状态：

第一步：一键删除所有容器和镜像
请在终端依次运行以下命令（直接复制粘贴）：

Bash

```
# 1. 强制停止并删除所有容器
# (如果有报错提示"requires at least 1 argument"说明本来就没容器，忽略即可)
docker stop $(docker ps -aq) 2>/dev/null
docker rm -f $(docker ps -aq)

# 2. 强制删除所有镜像
# (这一步最重要，必须把那些“幽灵镜像”删掉，否则 docker pull 永远不会下载新文件)
docker rmi -f $(docker images -q)

# 3. 清理残留网络和缓存
docker system prune -a -f
```
运行完后，再次输入 docker ps -a 和 docker images，应该看到列表是全空的。

第一步：配置 Docker Hub 镜像 (hub.rat.dev)
你需要编辑 Docker 的配置文件 daemon.json。

打开或创建配置文件：

Bash

```
sudo mkdir -p /etc/docker
sudo vim /etc/docker/daemon.json
```
写入以下内容： 如果文件是空的，直接复制粘贴；如果已有内容，请小心添加到现有 JSON 结构中。

JSON

```
{
  "registry-mirrors": [
    "https://hub.rat.dev"
  ]
}
```
(注意：hub.rat.dev 是第三方搭建的代理，稳定性可能不如大厂镜像，建议作为备选或在无法访问时使用。)

重载配置并重启 Docker：

Bash

```
sudo systemctl daemon-reload
sudo systemctl restart docker
```
验证是否生效： 输入 docker info，查看输出底部的 "Registry Mirrors" 字段：

Bash

`docker info`
如果看到 https://hub.rat.dev/，说明配置成功。

--------------
备份之后的更新：
一、解决 Docker 启动报错（必须配置）

你提到的 Docker 问题（Nginx 没跑起来、Calibre 没库）是因为你用了 nofail。 逻辑悖论： nofail 告诉系统“硬盘没挂载好也别管，继续开机”，于是系统就继续启动 Docker 了，但此时硬盘还没挂载上，Docker 自然就报错了。

你需要告诉 Docker：“虽然系统不因硬盘而停，但 Docker 你必须等硬盘。”

编辑 Docker 服务配置：
Bash

`sudo systemctl edit docker`
粘贴以下内容（强制依赖挂载点）： 注意：systemd 会把挂载路径中的 / 转换为 -。
ini
```

[Unit]
# 只有当这两块盘都挂载成功后，Docker 服务才会启动
Requires=mnt-disk_zhaoyang.mount mnt-disk_sony.mount
After=mnt-disk_zhaoyang.mount mnt-disk_sony.mount
```
生效配置：
Bash

`sudo systemctl daemon-reload`
做完这一步，哪怕 USB 硬盘识别慢了 20 秒，Docker 也会乖乖等 20 秒再启动，彻底根治重启容器才能恢复的问题。

二、“安全关机脚本”更新V7

> /mnt/disk_zhaoyang/006_software/001_os_images/nas_3865u/bye_disks_noplug.sh

三、关闭tv-backend的开发模式

找到了根源就好办了！

问题出在 Django 的开发模式。你的 tv-backend 容器目前默认运行的是 python manage.py runserver。 这个模式下，Django 会启动一个“文件监视器 (StatReloader)”，每秒钟扫描你挂载目录下的所有文件，看看有没有代码改动。 因为你挂载了 saveload_backend-master 代码目录，文件很多，所以它每秒钟都在疯狂读取 SSD，产生了那 5G 的日志。

解决方案：添加 --noreload 参数
你只需要在 docker-compose.yml 文件中，给 tv-backend 服务显式添加一行 command 指令，强制关闭自动重载功能。

1. 修改 docker-compose.yml

请编辑你的 yaml 文件，找到 tv-backend 部分，添加我下面高亮的那一行 (command: ...)：

YAML

  ```
# 1. Django 后端
  tv-backend:
    container_name: tv-backend
    build: ./saveload_backend-master
    restart: always
    ports:
      - "8001:8000"
    volumes:
      - ./saveload_backend-master:/app
      - ./vipdoc:/data/tdx/vipdoc:ro
    environment:
      - TZ=Asia/Shanghai
      - DATABASE_URL=postgres://creedxz:Rights_5743@db:5432/tradingview
      - TDX_DATA_ROOT=/data/tdx
    depends_on:
      - db
    # ▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼
    # 新增这一行！关闭自动重载，保护 SSD
    command: python manage.py runserver 0.0.0.0:8000 --noreload
    # ▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲
```
2. 应用更改并重建容器

修改保存后，在存放 docker-compose.yml 的目录下执行以下命令：

Bash

```
# 重新创建并启动容器 (Docker 会检测到配置变动)
docker-compose up -d tv-backend
```
3. 验证是否生效

容器重启好后，查看日志确认一下：

Bash

`docker logs --tail 20 tv-backend`
成功的标志：

以前的日志： 会显示 Watching for file changes with StatReloader。
现在的日志： 这句话消失了。只会显示 Starting development server at ...。

四、关闭fireflyiii、immich、vault warden的健康度检查

没问题，完全可以！

如果你的需求是**“只要它跑着就行，挂了等我发现了再手动处理，或者等它自己慢慢反应”**，那么设置为 3 小时一次是极好的省资源策略。

这样 Docker 一天只会读取硬盘检查它 8 次，对 SSD 的磨损几乎为零。

请直接使用下面的配置覆盖你原来的 `app` 和 `db` 部分：

```yaml
services:
  app:
    image: fireflyiii/core:latest
    hostname: app
    container_name: firefly_iii_core
    restart: always
    volumes:
      - firefly_iii_upload:/var/www/html/storage/upload
    env_file: .env
    networks:
      - firefly_iii
    ports:
      - 8011:8080
    depends_on:
      - db
    # --- 修改部分：极低频健康检查 ---
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/api/v1/health"]
      interval: 3h        # 改为每 3 小时检查一次
      timeout: 10s
      retries: 3
      start_period: 30s
    # -----------------------------

  db:
    image: mariadb:lts
    hostname: db
    container_name: firefly_iii_db
    restart: always
    env_file: .db.env
    networks:
      - firefly_iii
    volumes:
      - /DATA/docker_dbdata/firefly/db:/var/lib/mysql
    # 数据库也建议同步改为 3小时，否则它还会每几秒跳一次
    healthcheck:
      test: ["CMD", "healthcheck.sh", "--connect", "--innodb_initialized"]
      interval: 3h        # 改为每 3 小时检查一次
      timeout: 5s
      retries: 3

  # cron 服务部分保持不变...
  cron:
    image: alpine
    restart: always
    container_name: firefly_iii_cron
    env_file: .env
    command: sh -c "
      apk add tzdata && \
      ln -fs /usr/share/zoneinfo/$$TZ /etc/localtime && \
      echo \"0 3 * * * wget -qO- http://app:8080/api/v1/cron/$$STATIC_CRON_TOKEN;echo\"
      | crontab - && \
      crond -f -L /dev/stdout"
    networks:
      - firefly_iii
    depends_on:
      - app

volumes:
   firefly_iii_upload:
   firefly_iii_db:

networks:
  firefly_iii:
    driver: bridge
```

### 修改后的影响（必读）：

1.  **极度安静**：你的日志和磁盘读取将变得非常干净。
2.  **副作用**：如果 Firefly 在你刚做完检查后的第 1 分钟崩溃了，Docker **在接下来的 2小时59分钟内** 都不会发现它挂了，也就不会去重启它。服务会在这段时间内不可用。
    *   *对于个人记账来说，这完全可以接受。*

### 如何生效：
保存文件后执行：
```bash
docker compose up -d
```

这里有两种方法来验证：一种是“查户口”（看配置是否生效），另一种是“看疗效”（看日志是否安静了）。

### 方法一：最直观的方法——看“安静了没”

你之前运行那个 `docker events` 命令时，屏幕是疯狂滚动的。现在请再次运行同样的命令：

```bash
docker events --filter 'event=exec_create'
```

**判断标准：**
*   **以前**：大概每 5 秒钟就会跳出一行 `exec_create ... curl ...`。
*   **现在**：运行后，**盯着屏幕看 30 秒甚至 1 分钟，应该一行字都不会跳出来**。
    *   如果有内容跳出来，但不是 Firefly 的，说明生效了。
    *   如果完全没有任何输出，说明世界清静了。

*(按 `Ctrl + C` 退出)*

---

### 方法二：最精准的方法——查 Docker 内部配置

直接查看容器的元数据，看看 Docker 到底把间隔设为了多少。

运行这条命令：

```bash
docker inspect firefly_iii_core --format '{{json .Config.Healthcheck}}'
```

或者用这个更易读的命令（如果你没有安装 jq 工具，可能显示会比较乱，建议用上面那个或者 grep）：

```bash
docker inspect firefly_iii_core | grep -A 5 "Healthcheck"
```

**判断标准：**
你需要找到 **`Interval`** 这一项。
*   如果显示 **`10800000000000`** (单位是纳秒) 或者 **`3h0m0s`**。
*   **结论**：**设置成功生效！** (3小时 = 10800秒)。

*如果显示的还是 `30000000000` (30秒) 或者 `5s`，说明配置没更新，请重新运行 `docker compose up -d`。*

---

### 方法三：看容器状态

```bash
docker ps
```

**判断标准：**
查看 `firefly_iii_core` 的 `STATUS` 一栏。
*   刚启动时可能显示 `Up x minutes (health: starting)`。
*   过一会应该显示 `Up x minutes (healthy)`。
*   只要它没有变成 `Restarting`，且上述方法一验证安静了，那就是完美的。

**可以直接关掉！**

对于家用服务器，尤其是你自己就是唯一的管理员，出了问题你能手动重启，那么完全可以关掉健康检查来换取“绝对的硬盘宁静”。

这里提供两个方案，任你选择：

---

### 方案一：简单粗暴 - 彻底关闭 (推荐给强迫症)

修改你的 `docker-compose.yml`，在 `immich-server`, `redis`, `database` 下面都加上禁用指令。

**注意：**
1.  **Server**: 将原来的 `disable: false` 改为 `disable: true`。
2.  **Redis**: 这是一个独立的块，直接修改。
3.  **Database**: 原文件中没有写 healthcheck，说明它用了镜像默认的。我们需要显式添加一段来禁用它。

```yaml
services:
  immich-server:
    # ... 其他配置保持不变 ...
    restart: always
    healthcheck:
      disable: true       # <--- 改为 true，彻底关闭检查

  redis:
    container_name: immich_redis
    image: docker.io/valkey/valkey:8-bookworm@...
    restart: always
    # 直接禁用
    healthcheck:
      disable: true       # <--- 添加这行

  database:
    container_name: immich_postgres
    # ... 其他配置保持不变 ...
    restart: always
    # 数据库原本没写，加上这个来禁用默认检查
    healthcheck:
      disable: true       # <--- 添加这行
```

**生效方法：**
```bash
docker compose up -d
```

**后果：**
*   **优点**：Docker 再也不会去读硬盘检查它们了，绝对零 IO。
*   **缺点**：如果 Immich 卡死了（网页打不开但在后台进程还在），Docker 不会自动重启它，你需要手动发现并重启。

---

### 方案二：折中方案 - 3小时检查一次 (推荐给求稳派)

如果你担心万一数据库死锁了没人管，可以改为 3 小时一次。

```yaml
services:
  immich-server:
    # ...
    healthcheck:
      # Immich 官方镜像自带的检查命令
      test: ["CMD", "immich-healthcheck"]
      interval: 3h
      timeout: 10s
      retries: 3

  redis:
    # ...
    healthcheck:
      test: ["CMD-SHELL", "redis-cli ping || exit 1"]
      interval: 3h
      timeout: 5s
      retries: 3
      
  database:
    # ...
    healthcheck:
      # Postgres 标准检查命令
      test: ["CMD-SHELL", "pg_isready -U ${DB_USERNAME} -d ${DB_DATABASE_NAME}"]
      interval: 3h
      timeout: 5s
      retries: 3
```

### 总结建议

既然你已经解决了主要矛盾（Django 和 Firefly），对于 Immich 这种重型应用：

*   如果你追求 **极致的零读写**：选 **方案一（彻底关闭）**。Immich 很成熟，不容易无缘无故卡死。
*   如果你希望 **有一点兜底**：选 **方案二**。一天只读 8 次盘，对 SSD 寿命没有任何影响。

CasaOS 本质上是一个可视化的 Docker 管理面板。对于通过 CasaOS 应用商店（App Store）或者手动安装的应用，你有两种方法来修改配置。

这里教你最简单、最稳妥的方法：**直接在 CasaOS 的界面上修改（如果支持）** 或者 **通过命令行修改底层的 compose 文件**。

鉴于 CasaOS 的界面可能无法直接修改高级的 `healthcheck` 参数，最有效的方法是**导出配置 -> 修改 -> 重新导入/更新**。

### 方法一：通过 CasaOS 网页界面修改（最简单，但不一定所有版本都支持）

1.  打开 CasaOS 仪表盘。
2.  找到 **Vaultwarden** 应用图标。
3.  点击图标右上角的三个点 `...` -> **设置 (Settings)**。
4.  在弹出的窗口顶部，看看有没有 **"Export Docker Compose"** (导出 Docker Compose) 或者类似的按钮/选项。
    *   如果有，点击它查看当前的 YAML 配置。
    *   在配置里添加或修改 `healthcheck` 部分（代码见下文）。
    *   保存并应用。

*注意：很多版本的 CasaOS 设置界面只允许改端口和挂载卷，不支持直接加 healthcheck 代码。如果不行，请用方法二。*

### 方法二：通过命令行“手术刀”式修改（推荐，精准有效）

CasaOS 安装的应用其实都对应着一个 `docker-compose.yml` 文件。我们需要找到这个文件，修改它，然后让 Docker 重新加载。

**步骤如下：**

1.  **找到配置文件路径**
    CasaOS 的应用配置通常存储在 `/var/lib/casaos/apps/` 目录下。Vaultwarden 应该在这个目录下有一个对应的文件夹。
    
    打开 SSH 终端，输入：
    ```bash
    find /var/lib/casaos/apps -name "docker-compose.yml" | grep vaultwarden
    ```
    你应该会看到类似 `/var/lib/casaos/apps/vaultwarden/docker-compose.yml` 的路径。

2.  **编辑文件**
    使用 `nano` 或 `vi` 编辑这个文件：
    ```bash
    nano /var/lib/casaos/apps/vaultwarden/docker-compose.yml
    ```
    *(如果不确定路径，就用你刚才 find 命令查到的那个)*

3.  **加入健康检查代码**
    找到 `services:` 下面的 `vaultwarden` 部分，在 `image`, `volumes` 等同级的位置，加入以下代码：

    ```yaml
        # ... 其他原有配置 ...
        restart: always # 原有配置
        
        # --- 添加这部分 ---
        healthcheck:
          test: ["CMD", "/healthcheck.sh"]
          interval: 3h        # 改为 3 小时
          timeout: 10s
          retries: 3
        # -----------------
    ```
    *注意缩进！YAML 文件对缩进非常敏感，必须和上面的 `restart` 或 `image` 保持对齐。*

4.  **保存退出**
    *   按 `Ctrl + O` -> `Enter` (保存)
    *   按 `Ctrl + X` (退出)

5.  **让修改生效**
    CasaOS 有个机制，我们可以直接用 `docker compose` 命令让它生效。
    
    切换到该目录：
    ```bash
    cd /var/lib/casaos/apps/vaultwarden/
    ```
    
    重新启动应用（Docker 会自动检测配置变化并重建容器）：
    ```bash
    docker compose up -d
    ```

### 方法三：如果你在 CasaOS 界面找不到“导出配置”，也不想用命令行

你可以直接在 CasaOS 界面上 **卸载 Vaultwarden**，然后选择 **“自定义安装 (Custom Install)”**。

1.  在自定义安装界面，点击右上角的 **“Import” (导入)**。
2.  输入下面的配置（这是 Vaultwarden 的标准配置 + 3小时健康检查）：
    ```yaml
    name: vaultwarden
    services:
      vaultwarden:
        image: vaultwarden/server:latest
        container_name: vaultwarden
        restart: always
        volumes:
          - /DATA/AppData/vaultwarden:/data  # 注意：这里要改成你原本的数据路径！
        ports:
          - 80:80  # 注意：改成你原本用的端口
        healthcheck:
          test: ["CMD", "/healthcheck.sh"]
          interval: 3h
          timeout: 10s
          retries: 3
    ```
3.  **关键点**：一定要确保 `volumes` 里的路径是你之前用的路径，这样数据才不会丢。

### 总结
推荐使用 **方法二（命令行修改）**。它是最安全且无损的，只需修改一个文件，执行一条命令，立马生效，不需要担心数据路径配错的问题。