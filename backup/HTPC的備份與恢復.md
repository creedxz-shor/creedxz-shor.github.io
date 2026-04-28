備份：

分區信息
> /mnt/disk_zhaoyang/006_software/001_os_images/Digi002_Debian_Backup_20251228/disk_layout.txt

```
# 1. 切换 Root (你已经做了)
# sudo -I

# 2. 创建目录
mkdir -p /backup_temp

# 3. 定义变量 (注意：路径末尾不带斜杠)
BACKUP_DIR="/backup_temp"
DATE=$(date +%Y%m%d)

# 4. 执行备份
# 注意 --exclude 的顺序和写法
tar -cvpzf $BACKUP_DIR/desktop_debian13_backup_$DATE.tar.gz \
--exclude=$BACKUP_DIR \
--exclude=/proc --exclude=/sys --exclude=/dev --exclude=/run --exclude=/tmp \
--exclude=/mnt --exclude=/media \
--exclude=/lost+found \
--exclude=/home/creedxz/Alist \
--exclude=/home/creedxz/.cache \
--exclude=/home/creedxz/.local/share/Trash \
/
```

恢復：
启动进入 Live 系统：

启动虚拟机，进入 Live 桌面或命令行。
打开终端，切换到 root：sudo -i。
第二阶段：分区 (模拟原机结构)
这次我们要模拟的是你原机的结构（去掉 Windows，只保留 Debian）。根据你之前提供的 lsblk，原机 Debian 在 sda8，EFI 在 sda1。

我们不需要完全照搬 sda8 这种序号，简化为标准的 EFI + Root 结构即可。

运行分区工具：

Bash

cfdisk /dev/sda
选择 GPT 类型。
建立分区：

分区 1 (EFI)：New -> 512M -> Type 选 EFI System。
分区 2 (Root)：New -> 剩余所有空间 -> Type 选 Linux filesystem。
Write -> 输入 yes -> Quit。
格式化：

安装工具包：

Bash

apt install dosfstools

Bash

mkfs.vfat -F32 /dev/sda1
mkfs.ext4 /dev/sda2
挂载：

Bash

mount /dev/sda2 /mnt
mkdir -p /mnt/efi       # 注意！原机挂载点是 /efi，这里保持一致
mount /dev/sda1 /mnt/efi
第三阶段：拉取备份并解压
你需要把 NAS 里的备份文件弄进来。

挂载 NAS 共享 (SMB)：

Bash

# 1. 安装工具 (如果 Live 系统没有)
apt update && apt install cifs-utils

# 2. 挂载
mkdir -p /mnt/nas_source
# 假设 IP 是 31.138，共享名是 disk_sony
mount -t cifs -o username=admin //192.168.31.138/disk_sony /mnt/nas_source
# 输入 NAS 密码
解压 (恢复)： 假设备份文件在 /mnt/nas_source/desktop_backup.tar.gz：

Bash

# 这一步会花点时间
tar -xvpzf /mnt/nas_source/desktop_backup.tar.gz -C /mnt/
第四阶段：修正配置 (至关重要)
解压完不能直接重启，必须修正“身份证”(UUID) 和引导。

查看新 UUID：

Bash

blkid
记下 /dev/sda2 (Root) 和 /dev/sda1 (EFI) 的 UUID。
修改 fstab：

Bash

nano /mnt/etc/fstab
找到挂载 / 的那行，把 UUID 改成新硬盘 sda2 的。
找到挂载 /efi (或者原机的 /boot/efi) 的那行，把 UUID 改成新硬盘 sda1 的。
重要：如果 fstab 里有 swap 分区，请注释掉（加 #），因为虚拟机里我们没分 Swap，不注释会卡启动。
检查挂载点：确认 EFI 的挂载点写的是 /efi（配合你刚才建立的目录）。
```
# /etc/fstab: static file system information.
#
# Use 'blkid' to print the universally unique identifier for a
# device; this may be used with UUID= as a more robust way to name devices
# that works even if disks are added and removed. See fstab(5).
#
# systemd generates mount units based on this file, see systemd.mount(5).
# Please run 'systemctl daemon-reload' after making changes here.
#
# <file system> <mount point>   <type>  <options>       <dump>  <pass>
# / was on /dev/sda8 during installation
UUID=83128a6b-919a-44ad-ba53-9514cc1c41b5 /               ext4    errors=remount-ro 0       1
# swap was on /dev/sda9 during installation
#UUID=eddfb875-5297-48b1-b40d-8257e85237ee none            swap    sw              0       0
UUID=D76F-1131  /efi  vfat  defaults  0  1
```
重建目录：

Bash

# 补全被排除的目录
```
mkdir -p /mnt/proc /mnt/sys /mnt/dev /mnt/run /mnt/tmp /mnt/mnt /mnt/media
mkdir -p /mnt/home/creedxz/Alist  # 补全网盘挂载点
mkdir -p /mnt/home/creedxz/.cache
chmod 1777 /mnt/tmp
```
第五阶段：修复引导 (Grub)
Chroot 进入系统：

Bash

```
for i in /dev /dev/pts /proc /sys /run; do mount -B $i /mnt$i; done
chroot /mnt
```
安装 Grub：

```
apt install grub-efi-amd64
grub-install --target=x86_64-efi --efi-directory=/efi --bootloader-id=Debian /dev/sda
update-grub
```


exit
reboot
___________
进桌面了！ 这意味着系统核心已经恢复正常。

“所有程序都无法打开” 是一个非常典型的症状。 这通常不是因为 EFI 分区的问题，而是因为权限问题或者用户目录挂载问题。

最常见的原因是：/tmp 目录权限不对。

诊断与修复
Linux 下的很多图形程序（KDE, Gnome, Firefox等）启动时都需要往 /tmp 目录写临时文件。如果 /tmp 权限不够，程序就会闪退或点不开。

1. 检查并修复 /tmp 权限 (可能性 90%)

打开终端（如果图形终端打不开，按 Ctrl + Alt + F3 进入纯文本控制台，登录 root）。

执行命令：

Bash

ls -ld /tmp
正确的权限应该是：drwxrwxrwt (最后有个 t，这叫粘滞位)。
错误的样子：drwxr-xr-x (普通权限)。
修复命令：

Bash

chmod 1777 /tmp
2. 检查 /home/creedxz 的所有者 (可能性 50%)

你从备份包还原回来，有时候文件的所有者（Owner）可能会乱，导致你当前登录的用户（creedxz）读写不了自己的家目录配置。

检查命令：

Bash

ls -ld /home/creedxz
应该显示：drwxr-xr-x ... creedxz creedxz ...
如果显示 root root，那就坏了。
修复命令：

Bash

chown -R creedxz:creedxz /home/creedxz
3. 检查 /home/creedxz/.cache 是否存在

你在备份时排除了 /home/creedxz/.cache。 虽然理论上程序会自动创建它，但如果父目录权限不对，或者程序比较笨，可能会失败。

手动创建并修复：

Bash

mkdir -p /home/creedxz/.cache
chown -R creedxz:creedxz /home/creedxz/.cache
chmod 700 /home/creedxz/.cache
操作建议
请按 Ctrl + Alt + F3 切换到 TTY3 控制台，登录 root，一次性执行以下修复组合拳：

Bash

```
# 1. 修复 /tmp (最重要)
chmod 1777 /tmp

# 2. 修复家目录权限 (以防万一)
# 请确认你的用户名是 creedxz，如果不是请修改
chown -R creedxz:creedxz /home/creedxz

# 3. 补全缺失的缓存目录
mkdir -p /home/creedxz/.cache
chown creedxz:creedxz /home/creedxz/.cache

# 4. 彻底重启
reboot
```
結束。