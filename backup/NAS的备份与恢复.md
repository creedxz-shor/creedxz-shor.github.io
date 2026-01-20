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