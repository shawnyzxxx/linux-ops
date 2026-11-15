# 磁盘管理----zx

## 文章导读
本篇是 Linux 磁盘管理的练习，完整演示**逻辑卷（LVM）的创建、格式化、挂载、扩容与缩容**流程，涵盖 `pvcreate`、`vgcreate`、`lvcreate`、`lvreduce`、`lvextend` 等核心命令。

## 一、练习要求
1. 创建一个 5G 的逻辑卷 `mylv` 和 10G 的逻辑卷 `mylv2`
2. `mylv` 使用 ext4 文件系统，`mylv2` 使用 xfs 文件系统，分别挂载并创建文件
3. 将 `mylv` 缩小为 3G，将 `mylv2` 扩大为 13G

## 二、操作步骤
**1. 创建逻辑卷**
```bash
lvcreate -L 5G -n mylv myvg
lvcreate -L 10G -n mylv2 myvg
```
> `-L` 指定逻辑卷大小，`-n` 指定逻辑卷名称，`myvg` 为卷组名。

**2. 格式化逻辑卷**
```bash
mkfs.ext4 /dev/myvg/mylv
mkfs.xfs /dev/myvg/mylv2
```

**3. 创建挂载点并挂载**
```bash
mkdir /mnt/mylv
mkdir /mnt/mylv2
mount /dev/myvg/mylv  /mnt/mylv
mount /dev/myvg/mylv2 /mnt/mylv2
```

**4. 在挂载的逻辑卷上创建文件**
```bash
touch /mnt/mylv/testfile.ext4
touch /mnt/mylv2/testfile.xfs
```

**5. 调整逻辑卷大小**
先卸载，再调整：
```bash
umount /mnt/mylv
lvreduce -L 3G /dev/myvg/mylv      # 缩小 mylv 到 3G

umount /mnt/mylv2
lvextend -L 13G /dev/myvg/mylv2    # 扩大 mylv2 到 13G
```

**6. 重新挂载**
```bash
mount /dev/myvg/mylv  /mnt/mylv
mount /dev/myvg/mylv2 /mnt/mylv2
```

## 三、小结
LVM 的核心优势在于**在线调整卷大小**：扩容（`lvextend`）通常无需停机；而缩容（`lvreduce`）有风险，务必先卸载并确认文件系统支持缩小，必要时先备份数据。理解物理卷→卷组→逻辑卷的层级关系，是掌握磁盘管理的基础。