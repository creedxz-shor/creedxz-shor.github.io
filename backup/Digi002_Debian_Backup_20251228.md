文件位於win-nas盤（USB插頭標識ZY）

> /media/devmon/casadocker/003huchuan/Digi002_Debian_Backup_20251228

如果未来你重装了 Debian 13，只需要拿出备份的文件夹，按照以下步骤操作，5分钟内就能恢复到现在的完美状态。

1. 安装必要软件

```
# 更新系统
sudo apt update
# 安装核心软件
sudo apt install mpv rclone pipewire-jack libjack-jackd2-0 qpwgraph intel-gpu-tools
```
2. 恢复配置文件 (假设你在备份文件夹内)

```
# 恢复 MPV 配置
mkdir -p ~/.config/mpv
cp mpv.conf ~/.config/mpv/

# 恢复 Rclone 配置
mkdir -p ~/.config/rclone
cp rclone.conf ~/.config/rclone/

# 恢复 MPV 快捷方式
mkdir -p ~/.local/share/applications
cp mpv.desktop ~/.local/share/applications/

# 恢复系统服务 (需要 sudo)
sudo cp rclone-alist.service /etc/systemd/system/
sudo cp fuse.conf /etc/

# 重新加载服务
sudo systemctl daemon-reload
sudo systemctl enable --now rclone-alist.service
```
3. 最后的物理连线 打开视频，打开 qpwgraph，把 6 个点连上即可。
