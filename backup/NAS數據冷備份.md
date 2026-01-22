这是一个非常清晰的操作需求。既然你的环境是 x86 架构（性能较好），且已经明确了源盘（`sdc1`, `sdd5`）和目标盘（`sdb`），我们按照“**停服务 -> 重新分区格式化 -> 备份 -> 恢复服务**”的流程进行。

**⚠️ 高能预警：**
接下来的操作会**彻底清空** `/dev/sdb`（即标签为 `Xudong` 的那块盘）上的所有数据。请务必确认这块盘里的数据已经不再需要，或者已经备份。

---

### 第一步：停止 Docker 服务（必须）
为了确保 Immich、Firefly 等数据库文件不损坏，必须先停机。

```bash
# 停止 Docker
sudo systemctl stop docker
sudo systemctl stop docker.socket

# 再次确认所有容器已停止（应为空或报错）
docker ps
```

### 第二步：清理并格式化目标盘 (sdb)
我们要把 `sdb` (目前是 NTFS) 变成 `ext4`。

1.  **确保未挂载**
    虽然 `df -h` 没显示它挂载了，但为了安全，先执行一次卸载：
    ```bash
    sudo umount /dev/sdb1 2>/dev/null
    ```

2.  **重新分区 (使用 fdisk)**
    输入以下命令进入分区工具：
    ```bash
    sudo fdisk /dev/sdb
    ```
    *在 `fdisk` 交互界面中依次输入：*
    *   输入 `d` (删除现有分区，如果有多个分区，可能需要重复几次直到删光)
    *   输入 `n` (新建分区)
    *   输入 `p` (主分区)
    *   输入 `1` (分区号 1)
    *   直接按 **回车** (使用默认起始扇区)
    *   直接按 **回车** (使用默认结束扇区，占满全盘)
    *   *(如果提示是否删除原来的 NTFS 签名，输入 `Y` 确认删除)*
    *   输入 `w` (写入并退出)

### 3. 格式化并移除保留空间 (关键步骤)
使用 `-m 0` 参数来指定保留块百分比为 0。

```bash
# -m 0 : 设置保留空间为 0%
# -L BACKUP_1T : 设置卷标名字
# -T largefile : (可选优化) 如果你存的主要是大文件(电影/镜像)，加上这个参数可以减少inode数量，进一步释放一点点空间并加快格式化速度。如果是备份大量小照片，不要加这个参数。

# 通用推荐命令：
mkfs.ext4 -m 0 -L BACKUP_1T /dev/sdb1
```

### 第三步：挂载备份盘
创建一个专门的挂载点并挂载新的 ext4 分区。

```bash
sudo mkdir -p /mnt/backup_target
sudo mount /dev/sdb1 /mnt/backup_target
```

### 第四步：执行冷备份 (Rsync)
我们创建两个子目录分别存放两块源盘的数据，防止文件混淆。

**建议先进入 `tmux` 会话（防止 SSH 断开导致中断）：**
```bash
tmux
```

**1. 备份第一块盘 (zhaoyang / sdc1)**
源路径：`/mnt/disk_zhaoyang`
目标路径：`/mnt/backup_target/backup_zhaoyang`

```bash
sudo mkdir -p /mnt/backup_target/backup_zhaoyang
# 开始同步
sudo rsync -axvH --progress --delete /mnt/disk_zhaoyang/ /mnt/backup_target/backup_zhaoyang/
```

**2. 备份第二块盘 (disk_sony / sdd5)**
源路径：`/mnt/disk_sony`
目标路径：`/mnt/backup_target/backup_sony`

```bash
sudo mkdir -p /mnt/backup_target/backup_sony
# 开始同步
sudo rsync -axvH --progress --delete /mnt/disk_sony/ /mnt/backup_target/backup_sony/
```

> **参数解释：**
> *   `-a`: 归档模式，保留所有权限、时间戳（对数据库文件至关重要）。
> *   `-x`: 这个参数的作用是告诉 rsync：“只在这个文件系统内部跑，不要跨越到其他挂载点去”。(即 --one-file-system)
> *   `-v`: 显示详情。
> *   `-H`: 保留硬链接（Docker 镜像层可能用到）。
> *   `--delete`: **镜像模式**。如果源盘删除了文件，备份盘也会删除。如果你想要纯增量备份（保留源盘已删除的历史文件），请去掉这个参数。

### 第五步：验证与恢复
当两个 `rsync` 命令都跑完后：

1.  **检查空间占用：**
    ```bash
    df -h | grep /mnt/
    ```
    确认 `/mnt/backup_target` 的已用空间是否约为 670GB 左右（395G + 275G）。

2.  **卸载备份盘：**
    ```bash
    sudo umount /mnt/backup_target
    ```

3.  **恢复 Docker 服务：**
    ```bash
    sudo systemctl start docker
    ```

4.  **检查服务状态：**
    ```bash
    docker ps
    ```