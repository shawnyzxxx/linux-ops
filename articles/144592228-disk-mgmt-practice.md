# Linux 磁盘管理练习

## 文章导读
本篇是 Linux 磁盘管理的综合练习，涵盖**命令定位与移动、用户与家目录创建、LVM 逻辑卷的创建与挂载、交换空间配置、本地软件仓库搭建、自定义命令、Web 服务部署**等多个知识点，适合作为 Linux 系统管理的综合练手。

## 一、练习要求与参考命令

**1. 定位并移动 useradd 命令**
找到 `useradd` 命令文件，移动到 `/tmp` 目录并重命名为 `useradd_backup`：
```bash
which useradd
mv /usr/sbin/useradd /tmp/useradd_backup
```

**2. 创建带指定家目录的用户**
使用 `vim` 等命令创建用户 `usertest`，并以 `/usertest` 目录作为家目录：
```bash
mkdir /usertest
useradd -m /usertest usertest
```
> 注意：`-m` 会创建家目录；此处指定家目录路径时，应确保目录已存在。

**3. 创建逻辑卷并挂载（重启有效）**
将 `/dev/sda` 和 `/dev/nvme0n2` 加入卷组 `vgtest`，创建逻辑卷 `lvtest`，格式化为 ext4 并挂载到 `/usertest`：
```bash
pvcreate /dev/sda /dev/nvme0n2
vgcreate vgtest /dev/sda /dev/nvme0n2
lvcreate -n lvtest -L +5g vgtest
mkfs.ext4 /dev/vgtest/lvtest
mount /dev/vgtest/lvtest /usertest
```
> 要实现**重启有效**，需写入 `/etc/fstab`：`/dev/vgtest/lvtest /usertest ext4 defaults 0 0`。

**4. 创建第二个逻辑卷并挂载（重启有效）**
将 `/dev/sdb` 和 `/dev/nvme0n3` 加入卷组 `vgexam`，创建逻辑卷 `lvexam`，挂载到 `/iso`：
```bash
pvcreate /dev/sdb /dev/nvme0n3
vgcreate vgexam /dev/sdb /dev/nvme0n3
lvcreate -n lvexam -L +5g vgexam
mkfs.ext4 /dev/vgexam/lvexam
mount /dev/vgexam/lvexam /iso
```
> 同样写入 `/etc/fstab` 实现重启自动挂载。

**5. 创建交换空间**
在卷组 `vgexam` 中创建逻辑卷 `lvswap`，配置为交换空间并开机自动启用：
```bash
lvcreate -n lvswap -L +2g vgexam
mkswap /dev/vgexam/lvswap
swapon /dev/vgexam/lvswap
```
> 在 `/etc/fstab` 中加入 `/dev/vgexam/lvswap swap swap defaults 0 0` 实现开机自动启用。

**6. 搭建本地软件仓库**
将 rhel9.3 镜像中的 `AppStream` 和 `BaseOS` 目录复制到 `/iso` 目录，并配置为系统默认的本地软件仓库：
```bash
# 复制镜像目录
cp -r /run/media/.../AppStream /iso/
cp -r /run/media/.../BaseOS /iso/

# 创建仓库配置文件
cat > /etc/yum.repos.d/local.repo <<'EOF'
[AppStream]
name=local AppStream
baseurl=file:///iso/AppStream
enabled=1
gpgcheck=0

[BaseOS]
name=local BaseOS
baseurl=file:///iso/BaseOS
enabled=1
gpgcheck=0
EOF

yum clean all && yum makecache
```

**7. 创建自定义命令 testcmd**
要求执行 `testcmd testfile` 时，在当前目录创建 `testfile` 文件：
```bash
mkdir -p /usr/local/bin
cat > /usr/local/bin/testcmd <<'EOF'
#!/bin/bash
touch "$1"
EOF
chmod +x /usr/local/bin/testcmd

# 测试
testcmd testfile
ls testfile
```

**8. 安装并配置 httpd 服务**
安装 `httpd` 并配置开机启动，同时禁用 `firewalld` 防火墙：
```bash
yum install httpd -y
systemctl enable --now httpd
systemctl disable --now firewalld
```

## 二、小结
本次练习串联了 Linux 磁盘管理与系统服务配置的多个核心技能：LVM 的「物理卷→卷组→逻辑卷」层级关系、挂载与开机自动挂载（`/etc/fstab`）、交换空间配置、本地仓库搭建、自定义命令与服务管理。掌握这些内容，就具备了独立完成系统存储与基础服务部署的能力。