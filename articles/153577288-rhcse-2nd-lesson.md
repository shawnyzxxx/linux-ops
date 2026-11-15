# RHCSE 第二课：计划任务详解

## 文章导读
计划任务是 Linux 运维中实现自动化的核心手段，分为**一次性任务（at）**和**周期性任务（crond/crontab）**两大类。本文系统讲解两类计划任务的实现原理、命令用法、时间格式、黑白名单权限控制，以及 run-parts 扩展，适合需要掌握 Linux 定时任务的运维学习者。

## 一、一次性计划任务（at）

### 1. at 命令的特点
`at` 用于安排**一次性任务**，核心特点：
- **一次性任务**：任务执行一次后即结束
- **精确时间执行**：任务在指定确切时间执行，若系统在该时间未运行，任务不会执行
- **不支持补执行**：与 crontab 不同，`at` 不会在系统启动后检查并执行之前错过时间点的任务

**实现程序**：`atd`
`atd` 是 at 守护进程，在系统后台运行，负责监控 at 队列并执行到达预定时间的任务。

```bash
# 查找命令由哪个软件包提供
[root@localhost ~]# yum whatprovides */at
[root@class2 ~]# yum install at-3.1.23-11.el9.x86_64
# 查看软件版本
[root@localhost ~]# at -V
# 查看服务名字
[root@localhost ~]# rpm -ql at
/usr/lib/systemd/system/atd.service
```

### 2. at 命令一般用法
```bash
at [HH:MM] [yyyy-mm-dd]        # 设置计划任务
```

| 选项 | 说明 |
|---|---|
| `-l` | 列出系统上所有用户的 at 任务，等价于 `atq` |
| `-d` | 删除或取消一个任务 |
| `-v` | 使用时间格式，列出 at 任务 |
| `-c` | 列出任务之后的命令内容 |
| `-f` | 从文件中读取 |

```bash
# 设定任务
[root@server ~]# echo "echo Hello, World!" | at now + 5 minutes
[root@server ~]# at -f /file midnight
[root@localhost ~]# at 10:44
warning: commands will be executed using /bin/sh
at> ls -al ~ | tee /a.txt
at> <EOT>       # ctrl + d 生成任务
job 1 at Sun Jan  5 10:44:00 2025
```

> 修改任务的执行时间和任务内容：删除任务、重新创建。

```bash
# 查看任务有哪些
at -l
atq
# 查看任务的具体内容
at -c 14
# 删除任务
atrm 1
at -d 5
# 查看系统是否执行
tail -f /var/log/cron
watch -n 1 ls -l /root
```

**方法1：交互式设置计划任务**
```bash
[root@localhost ~]# at 4:30 2025-01-08
warning: commands will be executed using /bin/sh
at> echo 124 > /xixhaha
at> <EOT>
job 3 at Wed Jan  8 04:30:00 2025

# 列出计划任务
[root@localhost ~]# at -l
2       Tue Jan  7 11:14:00 2025 a root
3       Wed Jan  8 04:30:00 2025 a root
# 查看计划任务里面的内容
[root@localhost ~]# at -c 2
# 删除任务
[root@localhost ~]# atrm 3
```

**方法2：非交互式（文件）**
```bash
[root@localhost ~]# cat /file1
ls -a /root > /file.txt
ls /
[root@localhost ~]# at -f /file1 now +1 min
warning: commands will be executed using /bin/sh
job 5 at Mon Jan  6 11:31:00 2025
```

**注意事项：**
- `at` 命令的输出结果（标准输出和标准错误输出）不会显示到屏幕上
- `at` 中执行的命令最好使用**绝对路径**，不容易报错
- 过程：输入 `at 时间` → 回车 → 输入执行的命令 → `ctrl+d` 退出 `at`
- `at` 命令只会执行一次，时间过了就不再执行

**任务保存与日志位置：**
- 计划任务保存在目录：`/var/spool/at`
- 计划任务的日志：`/var/log/cron`
> 注意：如果执行的任务需要终端显示，该任务的输出不会显示在标准输出。

### 3. 时间的书写格式

**指定时间**
- `am`、`pm`：12 小时制
- `HH:MM`：24 小时制
- 年月日：`06122024`、`2024-12-06`、`12/06/2024`、`06.12.2024`
- 格式：`MMDD[CC]YY`、`[CC]YY-MM-DD`、`MM/DD/[CC]YY`、`DD.MM.[CC]YY`、`[HH:MM] [yyyy-mm-dd]`

**模糊时间**
`noon`（12:00）、`midnight`（00:00）、`teatime`（4pm）

**相对时间**
`now +1 min`、`now +2 days`

```bash
# 三天后的下午4点运行作业
at 4pm + 3 days
# 7月31日上午10点运行作业
at 10am Jul 31
# 明天的凌晨1点运行作业
at 1am tomorrow
```

### 4. 黑白名单
超级用户可以在任何情况下使用这些命令。对于其他用户，使用 `at` 的权限由 `/etc/at.allow` 和 `/etc/at.deny` 确定：
1. 格式：每行一个用户名称，不允许有空白字符
2. 系统先检测 `at.allow`，如果 `at.allow` 存在，其中的用户允许 `at`，即使 `at.deny` 也存在该用户也会被忽略
3. 如果 `at.allow` 不存在才检测 `at.deny`，其中用户不允许 `at`，剩下的用户可使用 `at`
4. `at.allow`、`at.deny` 都不存在，只有 root 可用 `at`

### 5. 权限注意点
不同用户对系统的使用权限不一样。**设置任务时，任务一定要在该用户的权限范围之内**：

```bash
[redhat@localhost ~]$ touch /root/1234
touch: 无法创建 '/root/1234': 权限不够

[redhat@localhost ~]$ at now +1 min
warning: commands will be executed using /bin/sh
at> touch /root/1234
at> <EOT>
job 11 at Mon Jan  6 11:47:00 2025
[redhat@localhost ~]$ at -l
11      Mon Jan  6 11:47:00 2025 a redhat
```
> 效果：显示任务 11 已被执行，但文件 `/root/1234` 实际不存在——因为 redhat 用户没有在 `/root` 下创建文件的权限。

### 6. at 练习
```bash
# 1、在未来的1分钟在/tmp创建一个文件
echo "touch /tmp/newfile.txt" | at now + 1 minute

# 2、在10点20分时在所有终端打印显示 XXXXXXXXXXXXXXXXXX
echo "wall 'XXXXXXXXXXXXXXXXXX'" | at 10:20

# 3、今天下午14:30列出当前目录的内容，并且将内容放到 a.txt 文件
echo "ls . > a.txt" | at 14:30
```

## 二、周期性计划任务（crond/crontab）

### 1. 概述
- **实现程序**：`crond`
- `crond` 守护进程在系统后台运行，负责检查 crontab 文件以确定哪些任务需要运行

```bash
# 找程序、程序的配置文件
[root@server ~]# systemctl status crond.service
[root@localhost ~]# rpm -ql cron
cronie          cronie-anacron  crontabs
[root@localhost ~]# rpm -ql crontabs
/etc/cron.daily
/etc/cron.hourly
/etc/cron.monthly
/etc/cron.weekly
/etc/crontab
```

### 2. 设定任务的方法

**方法1：`/etc/crontab`**
系统的 crontab 文件，由 crond 进程管理。

**方法2：`crontab -e -u root`**
以 root 身份添加计划任务，并以用户身份命名的文件保存在 `/var/spool/cron/` 目录下：

```bash
[root@class2 ~]# ll /var/spool/cron/
总用量 8
-rw-------. 1 redhat redhat 44 10月 18 11:38 redhat
-rw-------. 1 root   root   41 10月 18 11:31 root
[root@class2 ~]# cat /var/spool/cron/redhat
* * * * *    tree > /home/redhat/file.con

[root@class2 ~]# cat /var/spool/cron/root
* * * * *   echo "$(date)"  >> /file.cron

[root@localhost ~]# crontab -e      # 可以语法检查
[root@localhost ~]# crontab -u redhat -e
```

**常用命令：**
- `crontab -l`：列出当前用户的 crontab 文件内容
- `crontab -r`：删除当前用户的 crontab 文件

```bash
[root@localhost ~]# ll /var/spool/cron/
总用量 4
-rw-------. 1 root root 41 10月 13 11:02 redhat
[root@localhost ~]# cat /var/spool/cron/redhat
* 11 * * *    touch /home/redhat/file333
```

**修改、查看、删除任务**
```bash
# 修改任务的执行时间和任务内容
crontab -e
# 查看任务有哪些
crontab -l
# 默认查看 root 用户的周期任务
[root@localhost ~]# crontab -l
* * * * *  touch /file111111111111111
# 查看指定用户的周期任务
[root@localhost ~]# crontab -l -u redhat
* 11 * * *    touch /home/redhat/file333
# 查看系统周期任务的具体内容
cat /etc/crontab
# 删除指定用户的周期任务
[root@localhost ~]# crontab -r -u redhat
[root@localhost ~]# crontab -l -u redhat
no crontab for redhat
# 删除 root 用户的周期任务
[root@localhost ~]# crontab -r
[root@localhost ~]# crontab -l
no crontab for root
```
> 注意：查看系统周期任务只能查看 `/etc/crontab`，`crontab -l` 是查看用户周期任务。

**方法3：`/etc/cron.d/`**
包含为不同用户存储的不同类别的任务，系统定时任务的目录，由 crond 进程管理。创建一个命名清晰的文件（如 `nginx_maintenance`），可以快速找到和修改相关定时任务配置：

```
[root@localhost ~]# /etc/cron.d/nginx_maintenance
# 清理旧的日志文件，每周执行一次
0 2 * * 7 root /usr/local/bin/cleanup_nginx_logs.sh
# 检查配置文件的完整性，每天执行一次
0 1 * * * root /usr/local/bin/check_nginx_config.sh
```

**日志**：crontab 执行的每一项工作都会被记录到日志 `/var/log/cron`：
```bash
tail -f /var/log/cron
```
> `%` 需要转义。例如在 `/etc/crontab` 中使用 `date` 命令时，需要写成 `date +%R` 并对 `%` 做转义处理。

### 3. 时间的书写格式（crontab）

**特殊符号**

| 符号 | 说明 | 示例 |
|---|---|---|
| `*` | 代表指定时间单位中的每一个时刻 | `*` 在分钟字段表示每分钟 |
| `,` | 分隔列表中的项目 | `1,3,5 * * * *` 每小时第 1、3、5 分钟执行 |
| `-` | 定义范围 | `0 9-17 * * *` 早 9 点到晚 5 点每小时开始执行 |
| `/数字` | 指定步长 | `*/15 * * * *` 每 15 分钟执行一次 |

**字段顺序**：`分 时 日 月 周`

| 配置 | 含义 |
|---|---|
| `*/2 */2 */2 */2 *` | 分、时、日、月每 2 个单位执行 |
| `* * * * */2` | 每 2 个星期执行 |
| `* * L 1-6/2 *` | 每月 1-6 号中每 2 天，取最后一天 |
| `* 8 * * *` | 每天 8 点的每一分钟 |
| `0 8 * * *` | 每天 8:00 执行 |
| `13 0 1 * *` | 每月 1 日 00:13 执行 |
| `*/2 12 * * *` | 每天 12 点每两分钟执行 |
| `* 11,23 * * *` | 每天 11 点和 23 点的每一分钟执行 |
| `* 1-3 * * *` | 每天 1、2、3 点的每一分钟执行 |

**日期与星期冲突**：当日期和星期冲突时，两者**都会执行，是与关系**。例如 2025 年 1 月 5 日是星期日，满足日期和星期条件时任务都会执行。

**练习：每月 5、10、15、20 号，凌晨 0 点-6 点之间每隔两个小时执行**
```
* 0-6/2 5-20/5 * *
* 0,2,4,6 5,10,15,20 * *
```

**特殊日期写法**
- `L`：表示最后一个
- 每年 2 月 31 日执行某任务：由于日期不存在，任务不会运行
- 想在每年 2 月最后一天凌晨 2 点执行：`* * L 2 *`（每年二月最后一天，每个小时的每分钟，无论星期几）
- 日期和周对不上时：`* * L 2 7` 表示每个月 2 月最后一天（L）并且是星期日（7）时执行

**常见任务示例**
```bash
# 每天凌晨1点执行备份脚本
0 1 * * * /path/to/backup/script.sh
# 每周一下午4点执行某个任务
0 16 * * 1 /path/to/weekly/script.sh
# 每隔10分钟执行一个监控脚本
*/10 * * * * /path/to/monitoring/script.sh
# 每天11点11分执行 echo hello
11 11 * * * echo hello
```

**anacron**
`anacron` 能够在系统启动后自动执行定期任务，即使系统在任务预定时间未运行，任务也会在系统下次启动时执行：
```bash
[root@localhost ~]# rpm -ql cronie-anacron
/etc/anacrontab
```

### 4. cron 黑白名单
文件：`/etc/cron.allow`、`/etc/cron.deny`

- `cron.allow` 和 `cron.deny` 文件**不能用于限制 cron 的执行**，它们只限制 `crontab` 的使用。限制访问 crontab 对用户的现有 crontab 没有影响，其工作将继续直到 crontab 被删除。
- 调用 `crontab` 的用户必须能够读取 `cron.allow` 和 `cron.deny` 文件，否则它们被视为不存在。
- 当系统有 `/etc/cron.allow` 时，只有写入此文件的用户可以使用 `crontab` 命令
- 如果有 `cron.allow`，`cron.deny` 会被忽略，因为 `cron.allow` 优先级更高
- 当系统只有 `/etc/cron.deny` 时，写入此文件的用户不能使用 `crontab`，未写入的用户可以使用

**记录位置：**
- crontab 执行的每一项工作都会被记录到 `/var/log/cron`
- 用户新建工作后，该项工作会被记录到 `/var/spool/cron/` 目录

### 5. 扩展：run-parts
`run-parts` 可将后面接的「目录」内的所有文件找出来执行。

**步骤1**：假设有名为 `scripts` 的目录，包含可执行脚本：
```
/scripts/
├── script1.sh
├── script2.sh
└── script3.sh
```

**步骤2**：确保脚本有执行权限
```bash
chmod +x /scripts/*.sh
```

**步骤3**：设置周期性任务
```bash
[root@localhost ~]# vim /etc/crontab
# 添加：每天凌晨 1 点执行这些脚本
01 * * * *   run-parts  /scripts
```

实际示例：
```bash
[root@localhost ~]# mkdir /shell
[root@localhost ~]# vim /shell/1.sh
[root@localhost ~]# vim /shell/2.sh
[root@localhost ~]# vim /shell/3.sh
[root@localhost ~]# chmod u+x /shell/*
[root@localhost ~]# vim /etc/crontab
* 11 * * *   root   run-parts  /shell    # 注意 /shell 是目录
# 日志追踪
tail -f /var/log/cron
```

### 6. cron 练习
```bash
# 让 john 用户在指定的时间范围内执行 check.sh 脚本
0 11-17 * * * john /check.sh

# 每两个小时执行一次，且在凌晨 0 点-6 点之间，每月 5、10、15、20 号执行
0  0-6/2 5-20/5 * *

# 每周一、周三、周日，凌晨 2-6 点，每隔 30 分钟执行 echo backup >> /root/cron_back
0,30 2-6 * * 1,3,7   echo "backup" >> /root/cron_back
```

## 三、小结
计划任务是 Linux 自动化运维的基础。`at` 适合一次性、精确时间的临时任务，不支持补执行；`crond`/`crontab` 适合长期、周期性的重复任务，支持 `*`、`,`、`-`、`/` 等时间符号，并可通过 `/etc/cron.d/`、`run-parts` 灵活组织任务。配置时务必注意**权限范围**（任务需在用户权限内）和**黑白名单**规则，并通过 `/var/log/cron` 追踪任务执行。