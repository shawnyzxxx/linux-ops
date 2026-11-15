# RHCSE--SELinux

## 文章信息
- 原文发布于：2025-11-15 11:58:59
- 标签：linux、运维、服务器
- 原阅读量：1088

## 文章导读

本文面向 RHCSE 备考与 Linux 运维人员，系统讲解 **SELinux（Security-Enhanced Linux，安全增强型 Linux）** 的原理与实操。全文从 SELinux 的定义与价值讲起，对比它与传统 DAC 权限的差异，剖析“安全上下文”与“策略规则”两个核心概念，梳理 `/etc/selinux/` 下的配置文件结构与三种工作模式，再给出状态查看、模式切换、安全上下文管理、布尔值与端口标签等常用命令，最后通过三个 nginx 实验演示如何定位并解决 SELinux 导致的服务访问故障。文中命令、参数与输出示例均来自真实实验环境，可按步骤复现。

## 一、SELinux 的定义与核心价值

### 1.1 定义

SELinux（Security-Enhanced Linux，安全增强型 Linux）是集成到 Linux 内核中的**强制访问控制（MAC）**安全机制。它通过预先定义的“策略规则”限制进程（主体）对文件、端口等资源（客体）的访问，**即使进程拥有 root 权限，也只能执行被允许的操作**。

### 1.2 核心价值：弥补传统权限（DAC）的缺陷

传统 Linux 权限（读 / 写 / 执行、用户 / 组 / 其他）属于“自主访问控制（DAC）”：root 用户拥有无限权限，一旦进程被劫持（例如黑客入侵 Web 服务），攻击者就可以利用 root 权限肆意破坏。SELinux 属于“强制访问控制（MAC）”：即使是 root 进程，超出规则的操作也会被禁止。其核心价值体现在以下方面：

**1. 降低进程提权 / 越权风险**

- 场景：Web 服务器（如 Nginx）被黑客入侵，恶意进程以 nginx 用户身份运行。
- 无 SELinux：若 nginx 用户因配置失误拿到过高权限（如能访问 `/etc/passwd`、`/root` 目录），恶意进程可直接读取密码文件、篡改系统配置，甚至提权为 root。
- 有 SELinux：即使 nginx 用户有一般权限，SELinux 也会限制 `httpd_t` 域只能访问预设资源（如 `/var/www`），无法触碰系统敏感文件。

**2. 便于追溯恶意行为**

SELinux 会记录详细的访问审计日志（`/var/log/audit/audit.log`），包括“哪个进程、因什么策略被拒绝访问”。没有 SELinux 时，仅能通过一般权限日志（如 `/var/log/secure`）追踪用户操作，难以定位进程级的恶意访问（例如某个后台进程偷偷读取敏感文件）。

**3. 不依赖人工配置，容错率更高**

一般权限依赖管理员手动配置（如 `chmod`、`chown`），难免出现失误（如误设 777 权限）；一次配置失误就可能导致敏感资源暴露，且难以发现。有了 SELinux，即使一般权限配置不当，强制访问策略仍能兜底，降低配置失误的影响。

**4. 最小权限原则**

为每个进程分配“刚好够用”的权限（如 httpd 进程只能访问 Web 目录和 80 端口，无法读写 `/etc/passwd`），限制攻击面。

**5. 满足合规性要求**

满足金融、医疗等行业的安全合规标准（如 PCI DSS），这些场景必须启用强制访问控制机制。

## 二、SELinux 与传统 Linux 权限（DAC）的区别

### 2.1 关键对比

SELinux 与 Linux 一般权限（DAC，自主访问控制）是“双重防护”关系：一般权限管“谁（用户 / 组）能对资源（文件 / 目录）做什么（读 / 写 / 执行）”，SELinux 管“哪个进程能对哪个资源（按标签）做什么操作”。两者互补，且 **SELinux 优先级更高**；没有 SELinux 会失去“强制访问控制”的最后一道防线，恶意进程、权限溢出等风险大幅升高。

| 特性 | 一般权限（DAC，自主访问控制） | SELinux 权限（MAC，强制访问控制） |
| --- | --- | --- |
| 控制主体 | 以“用户 / 组 / 其他（u/g/o）”为核心，如 root 用户、www 组 | 以“进程（域）”和“资源（类型）”为核心，基于标签（上下文）控制 |
| 控制逻辑 | 资源所有者自主决定权限（如 `chmod 755` 开放读执行），谁拿到权限就能操作 | 系统强制按预设策略执行，即使文件权限是 777，进程无 SELinux 授权也无法访问 |
| 权限标识 | `r`（读）、`w`（写）、`x`（执行），用 `ls -l` 查看 | 进程标签（domain，如 `httpd_t`）、资源标签（type，如 `httpd_sys_content_t`），用 `ls -Z` 查看 |
| 优先级 | 低：只要 SELinux 禁止，即使一般权限允许也无法访问 | 高：覆盖一般权限，是最后一道防护 |
| 防护目标 | 防止“用户误操作”（如普通用户删系统文件）、“合法用户越权” | 防止“恶意进程提权”（如黑客入侵 Web 服务后访问 `/etc/passwd`）、“服务越权访问” |

### 2.2 实际访问流程：先过一般权限，再过 SELinux

当进程尝试访问资源（如 httpd 读 `/var/www/html/index.html`）时，系统按以下顺序校验：

1. **一般权限校验**：httpd 运行的用户（如 apache）是否有文件的读权限（`r`），无则直接拒绝。
2. **SELinux 校验**：httpd 进程的域（`httpd_t`）是否被允许访问文件的类型（`httpd_sys_content_t`），无则拒绝。

> **举例**：若 `/var/www/html/index.html` 权限是 777（一般权限允许所有人读），但 SELinux 策略禁止 `httpd_t` 访问该文件类型，httpd 仍无法读取。

## 三、SELinux 核心原理：安全上下文与策略规则

### 3.1 基本模型：主体、客体、策略、安全上下文

SELinux 的访问控制基于“主体（Subject）→ 客体（Object）→ 策略规则（Policy）”的三元模型：先通过“安全上下文”标识主体和客体的“身份”，再通过策略规则判断是否允许访问。

- **主体（Subject）**：正在运行的进程（如 httpd、sshd 进程），每个进程都有 SELinux 安全上下文。
- **客体（Object）**：被访问的资源（如文件、目录、端口、设备等），每个客体也有 SELinux 安全上下文。
- **策略（Policy）**：预定义的规则集合，规定“哪些主体（进程）可以访问哪些客体（资源）”，是 SELinux 的“大脑”。
- **安全上下文（Security Context）**：给主体和客体打上的“身份标签”，格式为 `user:role:type:level`（前三者是核心，level 用于 MLS 策略）。访问控制的核心是“比较主体和客体的安全上下文是否匹配策略规则”。

### 3.2 安全上下文：SELinux 的“身份标签”

安全上下文是 SELinux 最抽象也最关键的概念。以 `unconfined_u:object_r:httpd_sys_content_t:s0` 为例，各字段含义如下：

| 字段 | 含义 | 核心作用 | 示例（httpd 相关） |
| --- | --- | --- | --- |
| user（用户） | SELinux 用户（非 Linux 系统用户），标识“访问者类型” | 区分不同类型的访问主体（如系统进程、用户进程） | `unconfined_u`（不受限用户，如 root 进程）、`system_u`（系统进程） |
| role（角色） | SELinux 角色，关联“用户可执行的操作” | 限制用户 / 进程的“职责范围”（如系统角色、用户角色） | `object_r`（客体角色，用于文件 / 资源）、`system_r`（系统进程角色） |
| type（类型） | 最核心的字段，主体的 type 称为“域（domain）”，客体的 type 称为“类型” | 策略规则主要基于 type 定义（如 `httpd_t` 进程能否访问 `httpd_sys_content_t` 文件） | 主体 type：`httpd_t`（httpd 进程的域）；客体 type：`httpd_sys_content_t`（Web 目录的类型） |
| level（级别） | 用于 MLS（多级安全）策略，标识资源的“敏感程度”（如 `s0`、`s1:c0.c1023`） | 实现“高敏感资源只能被高权限进程访问” | 仅在 MLS 策略中生效，默认 targeted 策略不常用 |

主体（进程）能否访问客体（文件 / 端口等资源），取决于“主体的 type”与“客体的 type”之间是否存在允许访问的规则（allow 规则）：

- httpd 进程的 SELinux 类型是 `httpd_t`（主体 type），Web 目录 `/var/www/html` 的类型是 `httpd_sys_content_t`（客体 type）。
- SELinux 策略中预定义了规则：`allow httpd_t httpd_sys_content_t:file read;`（允许 httpd 进程读取 Web 目录的文件）。
- 若将 `/var/www/html` 的类型改为 `etc_t`（`/etc` 目录的类型），则 httpd 进程访问该目录时，SELinux 会拒绝（因为策略中没有 `allow httpd_t etc_t:file read;` 规则），即使 Linux 权限是 777 也无法访问。

### 3.3 SELinux 用户与角色

`/etc/selinux/targeted/contexts/users/` 目录下的 `guest_u`、`root`、`staff_u` 等文件，对应的是 **SELinux 用户（SELinux User）**——这是 SELinux 安全模型中用于身份标识的抽象概念，与 Linux 系统用户（如 `/etc/passwd` 中的用户）并非直接对应，而是通过映射关系关联（如 Linux 的 root 用户默认映射到 SELinux 的 root 用户）。

查看映射：用 `semanage login -l` 查看 Linux 用户对应哪个 SELinux 用户：

```bash
[root@rhce ~]# semanage login -l

Login Name           SELinux User         MLS/MCS Range        Service

__default__          unconfined_u         s0-s0:c0.c1023       *
root                 unconfined_u         s0-s0:c0.c1023       *
[root@rhce ~]#
```

系统可以识别以下七种 SELinux 用户：

```text
users
├── guest_u
├── root
├── staff_u
├── sysadm_u
├── unconfined_u
├── user_u
└── xguest_u
```

查看某个 SELinux 用户的默认上下文转换规则：

```bash
[root@rhce ~]# cat /etc/selinux/targeted/contexts/users/root
system_r:crond_t:s0		unconfined_r:unconfined_t:s0 sysadm_r:cronjob_t:s0 staff_r:cronjob_t:s0 user_r:cronjob_t:s0
```

含义：“当进程处于 `system_r:crond_t:s0` 这个初始上下文时，允许它转换到后面列出的目标上下文（即切换到新的 role 和 type）。”例如：`crond_t` 进程（定时任务进程）可以转换为 `unconfined_t`（无限制类型）或 `cronjob_t`（cron 任务专用类型），转换后以新的 type 运行。

#### 各 SELinux 用户的用途（典型场景）

每个 SELinux 用户文件定义了该用户在不同场景下的安全上下文转换规则，核心作用是限制该用户能使用的角色和权限范围：

| SELinux 用户 | 对应文件 | 典型用途 | 权限等级 |
| --- | --- | --- | --- |
| root | root | 对应 Linux 系统的 root 管理员，是 SELinux 中权限最高的用户，可切换到几乎所有角色（如 `sysadm_r`、`unconfined_r`），用于系统核心管理操作 | 最高（可执行敏感操作，如修改系统配置、管理服务） |
| unconfined_u | unconfined_u | 对应 Linux 中“无限制用户”（如普通用户但需较高权限），默认关联 `unconfined_r`（无限制角色），权限接近 root 但受部分限制，适合信任的普通用户 | 高（允许大部分操作，无需严格遵循 SELinux 类型限制） |
| sysadm_u | sysadm_u | 系统管理员专用 SELinux 用户，关联 `sysadm_r`（系统管理员角色），权限聚焦于系统管理（如启停服务、管理用户），但比 root 更受限（避免无差别高权限） | 中高（仅允许系统管理相关操作） |
| staff_u | staff_u | 对应“Staff 角色用户”（如中级运维人员），关联 `staff_r` 角色，权限介于管理员和普通用户之间，可执行部分管理操作（如 su 提权）但受严格限制 | 中等（有限的管理权限，核心操作需授权） |
| user_u | user_u | 普通受限用户，关联 `user_r` 角色，权限仅限于个人操作（如访问家目录、使用普通应用），无法执行系统管理操作，适合一般用户 | 低（仅允许个人级操作） |
| guest_u | guest_u | 临时访客用户，权限极低，仅允许最基础的操作（如浏览网页、读取公共文件），无法修改系统或个人配置，适合临时登录场景 | 极低（严格限制，防止任何敏感操作） |
| xguest_u | xguest_u | 图形界面下的访客用户，专为图形登录设计，权限与 `guest_u` 类似，但额外限制图形界面相关操作（如禁止修改桌面配置），适合公共电脑的临时使用 | 极低（图形场景的访客权限） |

#### SELinux 用户与角色（Role）的区别

SELinux 中，“用户”和“角色”是两个不同层级的安全概念，核心区别体现在“范围”和“关联关系”上：

| 维度 | SELinux 用户（User） | SELinux 角色（Role） |
| --- | --- | --- |
| 定义 | 抽象的身份标识，代表“谁在操作”（如 root、`user_u`） | 操作权限的“角色定位”，代表“扮演什么角色”（如 `sysadm_r`、`user_r`） |
| 核心作用 | 限制“允许使用的角色范围”（一个用户可关联多个角色，但不能超出其允许的范围） | 限制“允许访问的类型（Type）范围”（一个角色关联多个类型，类型决定实际访问权限） |
| 层级关系 | 上层概念，是角色的“容器”：用户 → 允许的角色 → 允许的类型 | 中层概念，连接用户和类型：角色是用户与类型之间的桥梁 |
| 举例 | `staff_u`（SELinux 用户）允许使用 `staff_r`、`user_r` 等角色，但不能使用 `sysadm_r`（除非授权） | `staff_r`（角色）允许访问 `staff_t`、`user_t` 等类型，但不能访问 `sysadm_t`（管理员专用类型） |
| 与 Linux 用户的关联 | Linux 用户需通过 `/etc/selinux/targeted/logins` 等文件映射到 SELinux 用户（如 Linux 的 root 映射到 SELinux 的 root） | 角色不直接与 Linux 用户关联，而是通过 SELinux 用户间接关联（用户→角色→类型） |

> **总结**：
> - SELinux 用户是“身份标签”，决定用户能“扮演哪些角色”；
> - 角色是“权限标签”，决定能“访问哪些类型的资源”；
> - 两者通过“用户→角色→类型”的层级关系，共同实现 SELinux 的“最小权限控制”：用户只能通过允许的角色，访问该角色允许的类型资源，避免权限滥用。
>
> 日常运维中，无需直接修改这些用户文件，而是通过 `semanage login` 命令管理 Linux 用户与 SELinux 用户的映射（如为普通 Linux 用户分配 `staff_u`），从而间接控制其权限范围。

### 3.4 SELinux 策略：规则的集合

策略是 SELinux 的“规则手册”。

#### 常见策略类型（RHEL 默认预装）

| 策略类型 | 适用场景 | 特点（教学重点） |
| --- | --- | --- |
| targeted | 默认策略，仅对“网络服务进程”（如 httpd、sshd、mysql）进行严格限制，对普通用户进程限制较少 | 兼顾安全与易用性，适合生产环境（99% 场景用此策略） |
| minimum | 在 targeted 基础上，仅限制“最关键的服务”（如 sshd），限制范围更小 | 适合对性能要求高、安全需求低的场景 |
| mls | 多级安全策略，基于“安全级别（level）”控制访问（如绝密文件只能被最高级进程访问） | 安全级别极高，适合军事、涉密场景，配置复杂 |

RHEL 9 中 SELinux 默认的策略是 **targeted**，如果要使用其他策略，需要额外安装相应的策略包：`selinux-policy-minimum`、`selinux-policy-mls`。

### 3.5 `/etc/selinux/` 配置文件结构

`/etc/selinux/` 目录的核心逻辑是“分层存储、按需加载”：

- 顶层文件（`config`、`semanage.conf`）控制全局模式；
- `targeted` 目录按“策略开关 - 上下文映射 - 核心策略”拆分配置，其中 `contexts/files` 是文件权限的关键，`policy/policy.33` 是访问规则的本体；
- 二进制文件（如 `.bin`、`policy.33`）用于 SELinux 高效加载，文本文件（如 `file_contexts`、`file_contexts.local`）用于用户配置修改。

日常运维中，修改最多的是 `config`（启用 / 禁用 SELinux）、`file_contexts.local`（自定义文件上下文），以及通过 `semanage` 命令间接修改 `logins`、`seusers` 等映射文件。

查看路径到类型的映射：

```bash
[root@shawn ~]# grep /usr/share/nginx/ /etc/selinux/targeted/contexts/files/file_contexts
/usr/share/nginx/html(/.*)?	system_u:object_r:httpd_sys_content_t:s0
```

#### 顶层核心文件（`/etc/selinux/` 目录下）

这两个文件是 SELinux 的“全局控制入口”，决定 SELinux 的整体运行模式和管理规则。

| 文件名 | 作用说明 |
| --- | --- |
| config | SELinux 主配置文件，控制 SELinux 的启用状态和策略类型，关键配置项包括：`SELINUX` 取值 `enforcing`（强制模式）、`permissive`（宽容模式）、`disabled`（禁用）；`SELINUXTYPE` 取值 `targeted`（默认，仅对关键服务生效）、`mls`（多级安全模式，更严格） |
| semanage.conf | `semanage` 命令（SELinux 策略管理工具）的配置文件，定义 `semanage` 的默认行为，如策略模块的存储路径、是否启用审计日志、默认的策略类型（与 config 中 `SELINUXTYPE` 对应） |

#### targeted 子目录（`/etc/selinux/targeted/`）

`targeted` 是 RHEL 9 默认的 SELinux 策略类型（对应 config 中 `SELINUXTYPE=targeted`），该目录存储 targeted 策略的所有具体配置和数据，是 SELinux 策略的核心存储区。

**1. 策略开关与映射文件（targeted 目录下直接文件）**

| 文件名 | 作用说明 |
| --- | --- |
| booleans.subs_dist | SELinux 布尔值（开关）的“分发版默认映射”文件，定义系统预装布尔值的默认名称和关联规则，用户一般无需修改（自定义布尔值需通过 `semanage boolean` 命令） |
| logins | SELinux 用户与 Linux 系统登录账号的映射配置文件，记录“哪个 Linux 账号对应哪个 SELinux 用户”（如 root 对应 root SELinux 用户，普通用户默认对应 `unconfined_u`） |
| setrans.conf | SELinux 翻译服务（setrans）的配置文件，用于将 SELinux 安全上下文（如 `unconfined_u:unconfined_r:unconfined_t:s0`）转换为人类易读的格式，主要用于日志显示和用户交互 |
| seusers | 与 `logins` 功能类似，是 SELinux 用户与 Linux 账号的兼容映射文件，优先级低于 `logins`，主要用于兼容旧版本 SELinux 配置（现代系统优先读取 `logins`） |

**2. contexts 子目录（`/etc/selinux/targeted/contexts/`）**

存储 SELinux 安全上下文的映射规则，定义“不同资源（文件、进程、端口、用户）应具备的安全上下文”，是 SELinux 访问控制的核心依据。

| 子文件 / 子目录 | 作用说明 |
| --- | --- |
| customizable_types | 定义“可自定义安全类型”的列表，即哪些 SELinux 类型（如 `httpd_sys_content_t`）允许用户通过 `chcon` 或 `semanage fcontext` 命令修改，避免用户误改系统关键类型 |
| dbus_contexts | 为 D-Bus 服务（Linux 桌面 / 系统的进程通信机制）定义安全上下文规则，确保 D-Bus 进程间的通信符合 SELinux 策略，防止未授权进程通过 D-Bus 访问敏感服务 |
| default_contexts | 定义“默认安全上下文”，包括用户登录时的默认上下文（如 `unconfined_u:unconfined_r:unconfined_t:s0`）、进程启动时的默认上下文，是 SELinux 上下文的“基础模板” |
| default_type | 指定系统默认的 SELinux 类型（`default_t`），用于“未明确匹配其他上下文规则”的资源（如临时文件、未知路径的文件），避免资源因无上下文而被 SELinux 直接拦截 |
| failsafe_context | “故障安全”上下文，当 SELinux 无法获取资源的正常上下文时（如配置错误、文件损坏），临时使用该上下文，确保系统不会因上下文缺失而完全无法运行 |
| files 子目录 | 文件安全上下文的核心配置目录，详见下方“files 子目录详解” |
| initrc_context | 为系统初始化进程（init 或 systemd 启动阶段的进程）定义默认安全上下文，确保初始化过程中进程的访问权限符合 SELinux 策略 |
| lxc_contexts | 为 LXC（Linux 容器）相关进程和资源定义安全上下文，限制容器内进程对宿主机的访问，实现容器与宿主机的 SELinux 隔离 |
| openssh_contexts | 为 OpenSSH 服务（ssh 进程、sshd 守护进程）定义安全上下文规则，确保 SSH 服务的访问（如密钥文件读取、端口绑定）符合 SELinux 策略 |
| removable_context | 为可移动存储设备（U 盘、移动硬盘）定义默认安全上下文，防止外部存储中的恶意文件通过 SELinux 权限漏洞访问系统敏感资源 |
| securetty_types | 定义“安全终端”（如 `/dev/tty1`、`/dev/console`）对应的 SELinux 类型，限制只有特定类型的终端才能执行敏感操作（如 root 登录） |
| sepgsql_contexts | 为 SE-PostgreSQL（PostgreSQL 数据库的 SELinux 增强）定义安全上下文，控制数据库进程对数据文件、端口的访问权限 |
| snapperd_contexts | 为 Snapper（Linux 系统快照工具）定义安全上下文，确保快照创建、恢复过程中对文件系统的操作符合 SELinux 策略 |
| systemd_contexts | 为 systemd 服务单元定义安全上下文规则，指定不同服务（如 `nfs-server.service`、`httpd.service`）启动时的默认进程上下文（如 `nfsd_t`、`httpd_t`） |
| userhelper_context | 为 userhelper 工具（辅助普通用户执行特权操作的工具）定义安全上下文，限制 userhelper 只能执行预设的特权操作，防止权限滥用 |
| users 子目录 | 存储不同 SELinux 用户（如 root、`staff_u`、`user_u`）的默认上下文配置，每个文件对应一个 SELinux 用户，定义该用户登录后的默认角色（r）、类型（t）等 |
| virtual_domain_context | 为虚拟化域（如 KVM 虚拟机）定义安全上下文，限制虚拟机进程对宿主机资源的访问，实现虚拟化环境的 SELinux 隔离 |
| virtual_image_context | 为虚拟机镜像文件（如 `.qcow2` 文件）定义默认安全上下文，防止镜像文件被未授权进程修改或读取 |
| x_contexts | 为 X Window 系统（Linux 图形界面）的进程和资源（如窗口、显示设备）定义安全上下文，确保图形界面的访问符合 SELinux 策略 |

**files 子目录详解（`/etc/selinux/targeted/contexts/files/`）**

该目录是文件 / 目录安全上下文的核心映射区，决定“特定路径的文件应具备哪种 SELinux 类型”（如 `/var/www/html` 对应 `httpd_sys_content_t`）。

| 文件名 | 作用说明 |
| --- | --- |
| file_contexts | 默认文件上下文映射文件，系统预装的“路径 - 类型”映射规则，覆盖绝大多数系统路径（如 `/etc` 对应 `etc_t`、`/home` 对应 `user_home_dir_t`），是文件上下文的“基础规则库” |
| file_contexts.bin | `file_contexts` 的二进制编译版本，由系统自动生成，SELinux 运行时优先加载二进制文件（加载速度更快），文本文件 `file_contexts` 是二进制文件的“源配置” |
| file_contexts.homedirs | 为用户家目录（如 `/home/user`、`/root`）定义默认上下文映射，细化家目录内不同子路径的类型（如 `/home/user/Documents` 对应 `user_home_t`） |
| file_contexts.homedirs.bin | `file_contexts.homedirs` 的二进制编译版本，作用同 `file_contexts.bin` |
| file_contexts.local | 用户自定义文件上下文映射文件，优先级高于 `file_contexts`，用户通过 `semanage fcontext` 命令添加的自定义规则会写入此文件，用于覆盖默认映射（如自定义 `/data/nfs` 对应 `nfs_t`） |
| file_contexts.subs | 文件路径“替换规则”文件，定义路径的简化别名（如将 `/var/run` 映射为 `/run`），确保 SELinux 能识别不同路径别名对应的上下文，避免因路径别名导致的规则失效 |
| file_contexts.subs_dist | `file_contexts.subs` 的“分发版默认文件”，系统预装的基础替换规则，用户自定义替换规则需修改 `file_contexts.subs`（而非此文件） |
| media 子目录 | 为媒体设备（如光盘、DVD）定义默认文件上下文，确保挂载媒体设备后，其内部文件的类型符合 SELinux 策略（如光盘文件对应 `iso9660_t`） |

**3. policy 子目录（`/etc/selinux/targeted/policy/`）**

存储 SELinux 核心策略文件，是 SELinux 访问控制规则的“二进制编译结果”。

| 文件名 | 作用说明 |
| --- | --- |
| policy.33 | SELinux targeted 策略的核心二进制文件，包含所有 SELinux 访问控制规则（如 `allow` 允许规则、`type_transition` 类型转换规则），是 SELinux 运行时加载的“策略本体”。后缀 33 是策略版本号，随系统更新可能变化（如 `policy.34`）；无法直接用文本编辑器查看，需通过 `sedispol` 工具解析为文本规则 |

### 3.6 工作模式：三种状态（控制严格程度）

SELinux 有三种工作模式，决定规则的执行强度，是必须掌握的基础：

| 模式 | 含义 | 适用场景 | 切换命令（临时） |
| --- | --- | --- | --- |
| Enforcing | 强制模式：严格执行 SELinux 策略，违反规则的操作会被拒绝并记录日志 | 生产环境（默认模式，必须启用） | `setenforce 1` |
| Permissive | 宽容模式：不拒绝违反规则的操作，但会记录日志（用于调试策略） | 新服务部署、排查 SELinux 相关问题时 | `setenforce 0` |
| Disabled | 禁用模式：完全关闭 SELinux（不加载策略，不记录日志） | 仅临时测试（不推荐生产环境，需重启生效） | 需修改 `/etc/selinux/config` 并重启 |

> **关键区别**：Permissive 是“记录不阻止”，Disabled 是“不记录不阻止”；Disabled 需重启才能切换到其他模式，而 Enforcing 和 Permissive 可通过 `setenforce` 实时切换。

## 四、SELinux 配置管理：基本操作

这部分是实操基础，需掌握“查看状态、修改模式、管理安全上下文、调整布尔值”等核心命令，为后续排错铺垫。

### 4.1 查看 SELinux 状态

```bash
# 1. 查看整体状态（模式、策略类型等）
[root@shawn ~]# sestatus
# 输出示例：
SELinux status:                 enabled
SELinuxfs mount:                /sys/fs/selinux
SELinux root directory:         /etc/selinux
Loaded policy name:             targeted
Current mode:                   permissive
Mode from config file:          enforcing
Policy MLS status:              enabled
Policy deny_unknown status:     allowed
Memory protection checking:     actual (secure)
Max kernel policy version:      33

# 2. 查看当前工作模式（简洁）
getenforce
# 输出：Enforcing / Permissive / Disabled
```

### 4.2 修改 SELinux 工作模式

**（1）临时修改（立即生效，重启后失效）**

```bash
[root@shawn ~]# setenforce 0  # 切换到 Permissive 模式（宽容）
[root@shawn ~]# setenforce 1  # 切换到 Enforcing 模式（强制）
```

**（2）永久修改（需重启生效，修改配置文件）**

```bash
[root@shawn ~]# vim /etc/selinux/config
# 核心配置：
# SELINUX=enforcing   # 永久启用强制模式（推荐）
# SELINUX=permissive # 永久启用宽容模式（调试）
# SELINUX=disabled   # 永久禁用（不推荐）

# 修改后需重启系统生效
reboot
```

### 4.3 安全上下文管理（核心操作）

**（1）查看安全上下文**

```bash
# 查看文件/目录的安全上下文
[root@shawn ~]# ls -Z /var/www/html
# 输出示例：-rw-r--r--. root root unconfined_u:object_r:httpd_sys_content_t:s0 index.html

# 查看进程的安全上下文（主体 type）
[root@shawn ~]# ps -Z | grep httpd
# 输出示例：system_u:system_r:httpd_t:s0    1234 ?        00:00:00 httpd
```

**（2）修改安全上下文（两种方式）**

方式 1：`chcon` 临时修改（重启或 `restorecon` 后失效）

```bash
# 格式：chcon [选项] 安全上下文 目标文件
# 示例：将 /data/web 的类型改为 httpd_sys_content_t（让 httpd 可访问）
chcon -t httpd_sys_content_t /data/web
```

方式 2：`semanage` 永久修改（写入策略，推荐）

`chcon` 是临时修改，`semanage` 会将修改保存到 SELinux 策略数据库，重启后不失效：

```bash
# 1. 安装 semanage 工具（默认可能未安装）
yum install policycoreutils-python-utils -y

# 2. 永久修改目录的类型（-a 新增，-t 指定类型，-r 递归）
semanage fcontext -a -t httpd_sys_content_t "/data/web(/.*)?"
# 注："/data/web(/.*)?" 表示匹配 /data/web 及所有子目录文件

# 3. 应用修改（让策略生效）
restorecon -Rv /data/web  # -R 递归，-v 显示详细信息
```

> **关键区别**：`chcon` 适合临时测试，`semanage` + `restorecon` 适合永久配置（生产环境必用）。

### 4.4 布尔值（Booleans）管理：快速调整策略开关

SELinux 预定义了大量“布尔值”（开关），用于快速启用 / 禁用特定规则（无需修改安全上下文），是解决服务访问问题的常用手段。

SELinux 布尔值（Boolean）是控制 SELinux 策略中“条件性规则”的开关变量，用于动态调整进程的权限范围。它们相当于“策略中的配置项”，通过开启（on）或关闭（off），可以在不修改核心策略的前提下，灵活允许或禁止特定场景的访问行为（如允许 httpd 访问网络、执行脚本等）。

### 4.5 端口标签管理：允许服务使用非标准端口

SELinux 不仅限制文件访问，还限制进程监听的端口（每个端口有“端口类型”标签）。例如，httpd 默认只能监听 `http_port_t` 类型的端口（如 80、443），若想让 httpd 监听 8080 端口，需将 8080 端口的类型改为 `http_port_t`。

```bash
# 1. 查看当前允许 httpd 监听的端口类型及端口
semanage port -l | grep http_port_t
http_port_t                    tcp      80, 443, 488, 8008, 8009, 8443

# 2. 永久添加 8080 端口到 http_port_t 类型
semanage port -a -t http_port_t -p tcp 8080

# 3. 移除端口标签（需 root 权限）
semanage port -d -t http_port_t -p tcp 8080

# 4. 验证添加结果
semanage port -l | grep http_port_t  # 此时会显示 8080
```

## 五、SELinux 实践操作：从“理论”到“解决问题”

这部分是教学重点，通过“服务访问故障排查”案例，掌握“SELinux 导致的问题如何定位和解决”，避免遇到问题就禁用 SELinux。

### 5.1 三步确认类型是否匹配

在 SELinux 中，“主体（如进程）的 type 类型”与“客体（如文件、端口）的 type 类型”是否“匹配”，本质是指 SELinux 策略中是否存在允许主体 type 访问客体 type 的规则（即 allow 规则）。若存在允许规则，则认为“类型匹配”（主体可合法访问客体）；若不存在，则“不匹配”（访问会被阻止）。

**步骤 1：获取主体（进程）的 type 类型**

主体通常是进程，通过 `ps -eZ` 命令查看进程的 SELinux 上下文，其中**第三个字段**即为主体的 type。

```bash
# 示例：查看 httpd 进程的 type
ps -eZ | grep httpd

# 输出类似：
system_u:system_r:httpd_t:s0  1234 ?  00:00:00 httpd
```

→ 主体（httpd 进程）的 type 是 `httpd_t`。

**步骤 2：获取客体（资源）的 type 类型**

客体可以是文件、目录、端口等，根据资源类型使用不同命令查看，第三个字段即为客体的 type。

| 客体类型 | 查看命令示例（获取 type） | 说明 |
| --- | --- | --- |
| 文件 / 目录 | `ls -Z /var/www/html` | 查看 `/var/www/html` 的 type |
| 端口 | `semanage port -l | grep http` | 查看 http 相关端口的 type |

```bash
# 示例：查看 /var/www/html 文件的 type
ls -Z /var/www/html

# 输出类似：
unconfined_u:object_r:httpd_sys_content_t:s0 index.html
```

→ 客体（文件）的 type 是 `httpd_sys_content_t`。

**步骤 3：检查策略中是否存在允许访问的规则**

通过 `sesearch` 工具（需安装 `setools-console` 包）查询 SELinux 策略中是否有允许“主体 type 访问客体 type”的规则。

核心语法：

```bash
sesearch -A -s 主体type -t 客体type -c 资源类别
```

参数说明：

- `-A`：仅显示 allow 规则（允许访问的规则）；
- `-s`：指定主体 type（如 `httpd_t`）；
- `-t`：指定客体 type（如 `httpd_sys_content_t`）；
- `-c`：指定资源类别（如 `file` 表示文件，`port` 表示端口，`process` 表示进程等）。

```bash
[root@shawn ~]# sesearch -A -s httpd_t -t httpd_sys_content-t
httpd_sys_content-t is not a valid type attribute
```

> **注意**：上面的命令中客体类型误写成了 `httpd_sys_content-t`（连字符），`sesearch` 因此报错“不是有效的 type 属性”。正确的写法应是 `httpd_sys_content_t`（下划线）。

### 5.2 allow 规则的通用格式与解析

SELinux 策略规则的通用格式为：

```text
allow 主体类型 客体类型:类别 { 权限列表 };
```

- **主体类型**：被允许操作的进程类型（通常对应某个服务，如 `httpd_t` 是 Apache 进程的类型）。
- **客体类型**：被访问的资源（文件 / 目录等）的 SELinux 类型（如 `httpd_sys_content_t` 是 Apache 网页内容的默认类型）。
- **类别**：资源的类型（如 `dir` 表示目录，`lnk_file` 表示符号链接文件）。
- **权限列表**：允许主体对客体执行的具体操作（如 `read` 表示读取）。

**第一条规则解析**：`allow httpd_t httpd_sys_content_t:dir { ioctl lock read };`

- 主体类型：`httpd_t` → 对应 Apache 服务进程（httpd 或 apache2）的 SELinux 类型。
- 客体类型：`httpd_sys_content_t` → SELinux 中专门用于标记“Apache 可访问的网页内容目录 / 文件”的类型（如 `/var/www/html` 目录默认类型）。
- 类别：`dir` → 表示客体是“目录”（而非文件或符号链接）。
- 权限列表：`{ ioctl lock read }`
  - `read`：允许读取目录内容（如列出目录下的文件）。
  - `lock`：允许对目录加锁（如防止并发操作冲突）。
  - `ioctl`：允许执行 I/O 控制命令（如获取 / 设置目录的底层属性）。
- 规则作用：允许 Apache 进程（`httpd_t`）对标记为 `httpd_sys_content_t` 的目录执行读取、加锁和 I/O 控制操作（确保 Apache 能正常访问网页目录结构）。

**第二条规则解析**：`allow httpd_t httpd_sys_content_t:lnk_file { getattr read };`

- 主体类型：`httpd_t`（同上，Apache 进程）。
- 客体类型：`httpd_sys_content_t`（同上，Apache 可访问的内容类型）。
- 类别：`lnk_file` → 表示客体是“符号链接文件”（软链接）。
- 权限列表：`{ getattr read }`
  - `read`：允许读取符号链接本身指向的目标路径。
  - `getattr`：允许获取符号链接的属性（如创建时间、权限等元数据）。
- 规则作用：允许 Apache 进程（`httpd_t`）读取标记为 `httpd_sys_content_t` 的符号链接文件，以及获取其属性（确保 Apache 能跟随网页目录中的软链接访问目标资源）。

### 5.3 实验 1：SELinux 端口限制实践

**步骤 1：确保 nginx 服务正在运行**

```bash
[root@shawn ~]# systemctl is-active nginx.service
active
```

**步骤 2：查看 nginx 进程的上下文**

```bash
[root@shawn ~]# ps -axZ | grep nginx
system_u:system_r:httpd_t:s0       3774 ?        Ss     0:00 nginx: master process /usr/sbin/nginx
system_u:system_r:httpd_t:s0       3775 ?        S      0:00 nginx: worker process
system_u:system_r:httpd_t:s0       3776 ?        S      0:00 nginx: worker process
unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023 4013 pts/1 S+   0:00 grep --color=auto nginx
```

**步骤 3：停止 nginx 服务并查看状态**

```bash
[root@shawn ~]# systemctl stop nginx.service
[root@shawn ~]# systemctl is-active nginx.service
inactive
```

**步骤 4：查看 SELinux 允许 nginx 侦听的端口**

```bash
[root@shawn ~]# semanage port -l | grep http
http_cache_port_t              tcp      8080, 8118, 8123, 10001-10010
http_cache_port_t              udp      3130
http_port_t                    tcp      80, 81, 443, 488, 8008, 8009, 8443, 9000
pegasus_http_port_t            tcp      5988
pegasus_https_port_t           tcp      5989
```

**步骤 5：修改 nginx 服务的监听端口为 82**

```nginx
[root@shawn ~]# vim /etc/nginx/nginx.conf
listen 82；
```

**步骤 6：再次启动 nginx 服务**

```bash
[root@shawn ~]# systemctl restart nginx.service
Job for nginx.service failed because the control process exited with error code.
See "systemctl status nginx.service" and "journalctl -xeu nginx.service" for details.
```

**步骤 7：查看 SELinux 日志**

```bash
[root@shawn ~]# grep "nginx" /var/log/audit/audit.log | grep "denied"
type=AVC msg=audit(1763035565.317:420): avc:  denied  { getattr } for  pid=3697 comm="nginx" path="/web/index.html" dev="dm-0" ino=34868111 scontext=system_u:system_r:httpd_t:s0 tcontext=unconfined_u:object_r:default_t:s0 tclass=file permissive=1
type=AVC msg=audit(1763035565.317:421): avc:  denied  { read } for  pid=3697 comm="nginx" name="index.html" dev="dm-0" ino=34868111 scontext=system_u:system_r:httpd_t:s0 tcontext=unconfined_u:object_r:default_t:s0 tclass=file permissive=1
type=AVC msg=audit(1763035565.317:421): avc:  denied  { open } for  pid=3697 comm="nginx" path="/web/index.html" dev="dm-0" ino=34868111 scontext=system_u:system_r:httpd_t:s0 tcontext=unconfined_u:object_r:default_t:s0 tclass=file permissive=1
type=AVC msg=audit(1763038243.675:436): avc:  denied  { name_bind } for  pid=4093 comm="nginx" src=82 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:reserved_port_t:s0 tclass=tcp_socket permissive=1
type=AVC msg=audit(1763038327.672:443): avc:  denied  { name_bind } for  pid=4116 comm="nginx" src=82 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:reserved_port_t:s0 tclass=tcp_socket permissive=1
type=AVC msg=audit(1763038354.752:449): avc:  denied  { getattr } for  pid=4122 comm="nginx" path="/web/index.html" dev="dm-0" ino=34868111 scontext=system_u:system_r:httpd_t:s0 tcontext=unconfined_u:object_r:default_t:s0 tclass=file permissive=1
type=AVC msg=audit(1763038354.752:450): avc:  denied  { read } for  pid=4122 comm="nginx" name="index.html" dev="dm-0" ino=34868111 scontext=system_u:system_r:httpd_t:s0 tcontext=unconfined_u:object_r:default_t:s0 tclass=file permissive=1
type=AVC msg=audit(1763038354.752:450): avc:  denied  { open } for  pid=4122 comm="nginx" path="/web/index.html" dev="dm-0" ino=34868111 scontext=system_u:system_r:httpd_t:s0 tcontext=unconfined_u:object_r:default_t:s0 tclass=file permissive=1
type=AVC msg=audit(1763038396.458:457): avc:  denied  { name_bind } for  pid=4157 comm="nginx" src=82 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:reserved_port_t:s0 tclass=tcp_socket permissive=0
```

**SELinux 阻止 Nginx 进程的原因**：Nginx 进程（类型 `httpd_t`）尝试绑定 82 端口（类型 `reserved_port_t`），但 SELinux 策略中没有“允许 `httpd_t` 对 `reserved_port_t` 类型的 `tcp_socket` 执行 `name_bind` 权限”的规则。具体来说：

- 82 端口属于 1-1024 特权端口，SELinux 默认将其标记为 `reserved_port_t` 类型；
- Nginx 进程的类型被错误标记为 `httpd_t`（正常应为 `nginx_t`），而 `httpd_t` 类型的进程默认没有绑定 `reserved_port_t` 端口的权限（`name_bind`）；
- 因此 SELinux 在强制模式下直接拒绝了绑定操作，导致 Nginx 无法启动或监听 82 端口。

**步骤 8：配置 SELinux 允许 nginx 侦听端口 82**

```bash
[root@shawn ~]# semanage port -a -t http_port_t -p tcp 82
# 添加 82 到 http_port_t
[root@shawn ~]# semanage port -l | grep http
http_cache_port_t              tcp      8080, 8118, 8123, 10001-10010
http_cache_port_t              udp      3130
http_port_t                    tcp      82, 80, 81, 443, 488, 8008, 8009, 8443, 9000
pegasus_http_port_t            tcp      5988
pegasus_https_port_t           tcp      5989
```

**步骤 9：重启 nginx**

```bash
[root@shawn ~]# systemctl restart nginx.service
```

### 5.4 实验 2：SELinux 类型限制实践——管理安全上下文

**步骤 1：临时修改 SELinux 模式为 Permissive**

```bash
[root@shawn ~]# setenforce 0
[root@shawn ~]# getenforce
Permissive
[root@shawn ~]#
```

**步骤 2：创建 nginx 主页面**

```bash
[root@shawn ~]# mkdir /web
mkdir: 无法创建目录 “/web”: 文件已存在
[root@shawn ~]# echo 123 > /web/index.html
```

**步骤 3：修改 nginx 配置**

```bash
[root@shawn ~]# vim /etc/nginx/nginx.conf
```

**步骤 4：重新加载配置并验证**

```bash
[root@shawn ~]# systemctl restart nginx.service
```

**步骤 5：修改 SELinux 模式为 Enforcing**

```bash
[root@shawn ~]# setenforce 1
[root@shawn ~]# getenforce
Enforcing
```

**步骤 6：查看 SELinux 日志**

```bash
[root@shawn ~]# systemctl restart nginx.service
[root@shawn ~]# curl 192.168.60.128:80
<html>
<head><title>403 Forbidden</title></head>
<body>
<center><h1>403 Forbidden</h1></center>
<hr><center>nginx/1.20.1</center>
</body>
</html>
[root@shawn ~]# grep "nginx" /var/log/audit/audit.log | grep "denied"
type=AVC msg=audit(1763177272.774:107): avc:  denied  { getattr } for  pid=4903 comm="nginx" path="/web/index.html" dev="dm-0" ino=34868111 scontext=system_u:system_r:httpd_t:s0 tcontext=unconfined_u:object_r:default_t:s0 tclass=file permissive=0
```

这条日志的关键是“**类型不匹配**”：

- 进程类型（`httpd_t`）和文件类型（`default_t`）都不符合 Web 服务的 SELinux 配置规范；
- 解决的核心思路是“让文件类型适配进程类型”（或修正进程类型），让 SELinux 策略中的允许规则生效。

日常排查时，只要看到 `avc: denied { getattr }` + 文件路径 + 类型为 `default_t`，**优先修正文件的 SELinux 类型**即可快速解决。

**步骤 7：修改 index.html 类型**

```bash
[root@shawn ~]# semanage fcontext -a -t httpd_sys_content_t /web/index.html
[root@shawn ~]# ll -Z /web/index.html
-rw-r--r--. 1 root root unconfined_u:object_r:default_t:s0 4 11月 15 11:22 /web/index.html
[root@shawn ~]# restorecon -R -v /web/index.html
Relabeled /web/index.html from unconfined_u:object_r:default_t:s0 to unconfined_u:object_r:httpd_sys_content_t:s0
```

**步骤 8：再次验证实验结果**

```bash
[root@shawn ~]# curl 192.168.60.128:80
123
```

### 5.5 实验 3：SELinux 类型限制实践（setroubleshoot 应用）

**步骤 1：安装、配置 nginx 的资源目录是 /home/redhat**

```bash
[root@shawn ~]# systemctl restart nginx.service
[root@shawn ~]# vim /etc/nginx/conf.d/test.conf
server {
        listen  84;
        root /home/redhat;
}
[root@shawn ~]# echo "home redhat" > /home/redhat/index.html
[root@shawn ~]# chmod 755 /home/redhat
[root@shawn ~]# chmod 644 /home/redhat/index.html
```

**步骤 2：重启 Nginx，发现失败**

```bash
[root@shawn ~]# systemctl restart nginx.service
Job for nginx.service failed because the control process exited with error code.
See "systemctl status nginx.service" and "journalctl -xeu nginx.service" for details.
[root@shawn ~]# grep "nginx" /var/log/audit/audit.log | grep "denied"
type=AVC msg=audit(1763177860.469:119): avc:  denied  { name_bind } for  pid=5049 comm="nginx" src=84 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:reserved_port_t:s0 tclass=tcp_socket permissive=0
```

**步骤 3：配置 SELinux 允许 nginx 侦听端口 84、修改目标资源类型**

```bash
[root@shawn ~]# semanage port -a -t http_port_t -p tcp 84
[root@shawn ~]# semanage fcontext -a -t httpd_sys_content_t "/home/redhat(/.*)?"
[root@shawn ~]# restorecon -R -v /home/redhat/
Relabeled /home/redhat from unconfined_u:object_r:user_home_dir_t:s0 to unconfined_u:object_r:httpd_sys_content_t:s0
Relabeled /home/redhat/.bash_logout from unconfined_u:object_r:user_home_t:s0 to unconfined_u:object_r:httpd_sys_content_t:s0
Relabeled /home/redhat/.bash_profile from unconfined_u:object_r:user_home_t:s0 to unconfined_u:object_r:httpd_sys_content_t:s0
Relabeled /home/redhat/.bashrc from unconfined_u:object_r:user_home_t:s0 to unconfined_u:object_r:httpd_sys_content_t:s0
Relabeled /home/redhat/index.html from unconfined_u:object_r:user_home_t:s0 to unconfined_u:object_r:httpd_sys_content_t:s0
[root@shawn ~]# systemctl start nginx
[root@shawn ~]# curl 192.168.60.128:84
<html>
<head><title>403 Forbidden</title></head>
<body>
<center><h1>403 Forbidden</h1></center>
<hr><center>nginx/1.20.1</center>
</body>
</html>
```

**下载并启用 setroubleshoot**

```bash
[root@shawn ~]# yum install setroubleshoot -y
[root@shawn ~]# systemctl start setroubleshootd.service
```

**步骤 4**

```bash
[root@shawn ~]# ausearch -m AVC -c nginx --raw | audit2allow -M nginx_access_home
******************** IMPORTANT ***********************
To make this policy package active, execute:

semodule -i nginx_access_home.pp

[root@shawn ~]# semodule -i nginx_access_home.
```

两条命令的含义：

- `ausearch -m AVC -c nginx --raw`：从 SELinux 审计日志（`/var/log/audit/audit.log`）中，筛选出与 Nginx 相关的“访问拒绝事件（AVC）”，并以原始格式输出。核心是“抓出 Nginx 被 SELinux 拦下来的操作”（比如“想读文件被拒”“想进目录被拒”）。
- `audit2allow -M nginx_access_home`：接收前一段的拒绝日志，自动分析“谁（Nginx 进程）需要什么权限”，生成两个文件：
  - `nginx_access_home.te`：策略源码文件（人类可读，记录具体允许规则）；
  - `nginx_access_home.pp`：编译后的二进制策略包（可直接加载到 SELinux）。
  - → 核心是“把‘被拒绝的操作’变成‘允许的规则’”。

**步骤 5**

```bash
[root@shawn ~]# curl http://192.168.60.128:84
home redhat
```

## 六、小结

SELinux 是 Linux 内核层面的强制访问控制（MAC）机制，与 DAC 一般权限构成“双重防护”，且优先级更高。理解 SELinux 的关键在于两点：一是**安全上下文**，即主体（进程域）与客体（资源类型）的“身份标签”，主体能否访问客体取决于两者之间是否存在 allow 规则；二是**策略与工作模式**，生产环境应保持 `Enforcing` + `targeted` 策略。遇到服务访问故障时，按“查主体 type → 查客体 type → 用 `sesearch` 查 allow 规则”三步定位，再通过 `semanage port`、`semanage fcontext` + `restorecon`、`audit2allow` 生成策略模块等方式放行，而不是直接禁用 SELinux。
