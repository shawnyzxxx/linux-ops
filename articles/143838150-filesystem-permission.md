# Linux文件系统权限管理

## 文章导读
本篇是 Linux 文件系统权限管理的练习，围绕**用户组、目录权限、SGID（组继承）**展开，讲解如何通过 `chmod`、`chgrp` 及特殊权限位实现精细化的目录访问控制。

## 一、练习要求
1. 创建 `g1` 组，创建一个属于 redhat 用户 `g1` 组的文件 `redhat.txt`
2. 新建 `/sc` 目录，所属组为 `group` 组；root 用户和 `group` 组用户可在其中创建文件，其他人无任何权限
3. 新建 `/cw` 目录作为财务部存储目录，仅财务部人员可写入，且财务部人员创建的文件自动属于 `mygroup` 组
4. 设置 `helen` 用户对 `/sc` 和 `/cw` 目录可读、可写、可执行
5. 设置 `/test` 目录为公共存储目录，对所有用户可读、写、执行，但用户只能删除属于自己的文件
6. 在 `/test/dir` 里创建的新文件自动属于 `temp` 组

## 二、参考命令
**1. 创建组与文件**
```bash
groupadd g1
usermod -aG g1 redhat
touch /root/redhat.txt
chgrp g1 /root/redhat.txt
```

**2. 配置 /sc 目录**
```bash
mkdir -p /sc
chgrp group /sc
chmod 770 /sc
```
> `770`：属主与属组有读写执行权限，其他用户无任何权限，满足「root 与 group 组可写，其他人无权限」。

**3. 配置 /cw 目录（财务部，SGID）**
```bash
mkdir -p /cw
chgrp mygroup /cw
chmod 2770 /cw
```
> `2` 即 SGID 特殊权限位：该目录下新创建的文件会**自动继承目录的属组**（mygroup），而不是创建者的主组，从而实现「财务部人员文件统一归属 mygroup」。

**4. 授权 helen 用户**
```bash
useradd helen
setfacl -m u:helen:rwx /sc
setfacl -m u:helen:rwx /cw
```
> 也可直接把 helen 加入对应组后通过组权限控制；`setfacl` 提供更细粒度的授权。

**5. 配置 /test 公共目录（sticky bit）**
```bash
mkdir -p /test
chmod 1777 /test
```
> `1` 即 sticky bit（粘滞位）：所有人都能读写执行，但**只能删除自己创建的文件**，防止互相删改。

**6. /test/dir 新文件自动属 temp 组（SGID）**
```bash
mkdir -p /test/dir
chgrp temp /test/dir
chmod 2770 /test/dir
```

## 三、小结
本篇集中练习了 Linux 权限管理的核心：普通权限位（`chmod`）、属组（`chgrp`）、以及两个特殊权限位——**SGID（2）** 实现组继承、**sticky bit（1）** 保护文件不被误删。掌握这两个特殊位，是做好共享目录权限管理的关键。