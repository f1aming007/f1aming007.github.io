# Linux文件系统btrfs


# 使用Btrfs 快照备份
[btrfs](https://wiki.archlinux.org/title/Btrfs)
https://wiki.archlinux.org/title/Btrfs
## 介绍
Btrfs 是 Linux 的现代写入复制 （COW） 文件系统，实现快照功能，可以瞬间创建快照，刚传家时几乎不占用任何空间，

<!--more-->

## btrfs 的工具
首先 Linux 下已经有了很多支持 btrfs 快照的备份工具了，比如 [Timeshift](https://github.com/teejee2008/timeshift), [Back In Time](https://backintime.readthedocs.io/en/latest/) 以及 [Snapper](https://wiki.archlinux.org/title/Snapper)

## 快照操作
* `btrfs subvol snapshot source destination`
    * 比如使用 `btrfs subvol snapshot / /snapshots/example`， 即可在/snapshots 创建一个为名为example 的快照， 除了不包含 / 里的子卷以外内容会与 / 相同， 快照的创建及户可以瞬间完成， 而且刚创建时几乎不占用任何空间， 
    * 使用`btrfs subvol snapshot -l ...` 可以创建一个只读快照
* `mv`
    * 没错，就是 mv 命令，当需要移动一个快照的位置时，只需要像对待文件夹一样使用 mv 命令。
* `btrfs subvol delete`
    * 例如 btrfs subvol delete /snapshots/example 可以删除前面建立的快照，如果直接用 Delete 删除则相应的硬盘空间无法得到释放。

    
## 发送到外部存储
将快照发送到外部存储
* `btrfs send`
* `btrfs receive`
*  这两条组合使用 `btrfs send /snapshots/example | btrfs receive /destination/` 会将 /snapshots/example 发送到 /destination/ 下
*  用 `btrfs send -p` 和 `btrfs receive` 来进行增量备份
*  `btrfs send` 只支持只读快照


## 列出子卷 
使用 `btrfs subvol list /` 可以列出 / 所在的 btrfs 分区的所有子卷（包括快照）
```bash
ID 270 gen 33666 top level 5 path snapshots
ID 317 gen 33718 top level 5 path @
ID 505 gen 27199 top level 270 path snapshots/auto-2018-07-27 01:00:00
ID 506 gen 27250 top level 270 path snapshots/auto-2018-07-27 02:00:00
```

## 创建存储快照的子卷
由于byrfs 的快照没办法分区创建，所以需要在每个需要备份的btrfs 分区建立一个用于存储快照的子卷
```bash
mount -o subvol=/ -U 71c631b0-e53f-40e4-aaac-22ae6fddbb96 /mnt
btrfs subvol create /mnt/snapshots
umount /mnt
```

> 分区的 UUID 可以通过 lsbk -f 查看


## 建立相应的文件夹
创建相应的文件夹， 比如我们需要分布备份root 和 home 两个btrfs 分区
```bash
mkdir -p /.snapshot/root
mkdir -p /.snapshot/home
```

## 把子卷挂载到相应的文件夹
最后把新建的子卷挂载到相应的文件夹，然后修改 /etc/fstab 添加相应的条目，之后重启或使用 mount -a 加载
```bash
UUID=71c631b0-e53f-40e4-aaac-22ae6fddbb96  /.snapshot/root/  btrfs subvol=snapshots,rw,noatime,compress=lzo,space_cache,commit=120,trim
UUID=3e80685a-cce5-425e-b478-3b1a008b9a48  /.snapshot/home/  btrfs subvol=snapshots,rw,noatime,compress=lzo,space_cache,commit=120,trim
```

## 自动快照
```bash
#!/bin/bash
set -Eeo pipefail

if [[ $EUID -ne 0 ]]; then
  echo "This script must be run as root" 
  exit 1
fi

SUB_0=("/" "/.snapshots/root")
SUB_1=("/home" "/.snapshots/home")
snapshots_config=(
  SUB_0[@]
  SUB_1[@]
)

dt=$(date '+%Y-%m-%d %H:%M:%S')
dt_now=$(date --date="$dt" +%s)
# Delete snapshots that older than 10 days
find "/.snapshots/" -maxdepth 2 -iname "auto-*" | while read file; do
  timestamp=$(echo "$file" | grep -Eo "[[:digit:]]{4}-[[:digit:]]{2}-[[:digit:]]{2} [[:digit:]]{2}:[[:digit:]]{2}:[[:digit:]]{2}")
  dt_file=$(date --date="$timestamp" +%s)
  let "tDiff=$dt_now-$dt_file"
  if [[ "$tDiff" -ge 864000 ]]; then
    btrfs subvol delete "$file"
  fi
done

snapshot_name="auto-$dt"
COUNT=${#snapshots_config[@]}
for ((i=0; i<$COUNT; i++))
do
  src=${!snapshots_config[i]:0:1}
  dst=${!snapshots_config[i]:1:1}
  # Create new snapshot
  btrfs subvol snapshot "${src}" "${dst}/$snapshot_name"
done
```

每次执行时会在 /.snapshots/root/ 和 /.snapshots/home/ 下分别创建两个格式为 auto-yyyy-mm-dd hh:mm:ss 的快照，同时删除 10 天以前的快照。
一个成熟的脚本应该学会自己按需触发，最简单的方式是修改 cronjob，缺点是很难进行检测和记录。因为脚本需要 sudo 权限，所以这里需要修改 root 的 cronjob，运行 sudo crontab -e 然后添加

```bash
# 每次启动时
@reboot /bin/bash /root/snapshot.sh
# 每隔一小时
@hourly /bin/bash /root/snapshot.sh
```

或者如果希望用 systemd 管理：
首先添加一个 systemd service，新建 /etc/systemd/system/snapshot.service:
```bash
[Unit]
Description=Create btrfs snapshot
After=syslog.target

[Service]
Type=oneshot
ExecStart=/bin/bash /root/snapshot.sh
StandardOutput=syslog
StandardError=syslog
SyslogIdentifier=snapshot

[Install]
WantedBy=multi-user.target
```

然后通过设置 systemd timer 来设置定时任务，新建 /etc/systemd/system/snapshot.timer:

```bash
[Unit]
Description=Create btrfs snapshot

[Timer]
# 开机启动 1 分钟后
OnBootSec=1min
# 每隔 1 小时
OnUnitActiveSec=1h

[Install]
WantedBy=timers.target
```
最后开启 timer 即可

```bash
systemctl enable --now snapshot.timer
```
## 手动快照

可以手动使用 btrfs subvol snapshot 来创建快照
```bash
#!/bin/bash
set -Eeo pipefail

if [[ $EUID -ne 0 ]]; then
  echo "This script must be run as root" 
  exit 1
fi

SUB_0=("/" "/.snapshots/root")
SUB_1=("/home" "/.snapshots/home")
snapshots_config=(
  SUB_0[@]
  SUB_1[@]
)

dt=$(date '+%Y-%m-%d %H:%M:%S')
dt_now=$(date --date="$dt" +%s)
snapshot_name="manual-$dt"
COUNT=${#snapshots_config[@]}
for ((i=0; i<$COUNT; i++))
do
  src=${!snapshots_config[i]:0:1}
  dst=${!snapshots_config[i]:1:1}
  # Create new snapshot
  btrfs subvol snapshot "${src}" "${dst}/$snapshot_name"
done
```

## 备份到外部存储
本地快照的主要作用是及时进行备份以防止意外操作，而长久可靠的备份还是需要保存到外部存储。使用 btrfs send 和 btrfs receive 可以将快照增量备份到外部存储，既节省了时间又节省了空间。
由于 btrfs send 只支持只读 (read-only) 子卷，所以我们需要用到 btrfs subvol snapshot -r ... 创建只读快照。
第一次创建只读快照

```bash
btrfs subvol snapshot -r /home /.snapshots/home/readonly && sync
```
然后发送到外部存储（第一次花费时间与用 cp 花费的时间相当）

btrfs send /.snapshots/home/readonly | btrfs receive /dev/sda/backup
以后只需要进行增量备份即可（仅仅需要复制更改过的文件）

```bash
btrfs subvol snapshot -r /home /.snapshots/home/readonly-new && sync
btrfs send -p /.snapshots/home/readonly /.snapshots/home/readonly-new | btrfs receive /dev/sda/backup
```
旧的只读快照可以删除

```bash
btrfs subvol delete /.snapshots/home/readonly
mv /.snapshots/home/readonly-new /.snapshots/home/readonly
btrfs subvol delete /dev/sda/backup/readonly
mv /dev/sda/backup/readonly-new /dev/sda/backup/readonly
```

当然建议是每次都把旧的本地只读快照删除，而外部存储的只读快照按照时间重命名，空间不够时再删除



