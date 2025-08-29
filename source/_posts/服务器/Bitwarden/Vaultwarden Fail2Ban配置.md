---
author: 竹林听雨
tags:
  - Fail2Ban
  - Vaultwarden
  - 转载
title: Vaultwarden Fail2Ban配置
published: 'true'
categories:
  - 服务器
  - Bitwarden
abbrlink: 1777f4d4
date: 2025-03-02 00:52:05
updated: 2025-08-30 00:56:05
---
[1.强化指南 |  Wiki 中文版](https://rs.ppgg.in/configuration/security/hardening-guide)
# 官方原版

对应的[官方页面地址](https://github.com/dani-garcia/vaultwarden/wiki/Fail2Ban-Setup)
[2.Fail2ban 设置 | Vaultwarden Wiki 中文版](https://rs.ppgg.in/configuration/security/fail2ban-setup#setup-for-web-vault)
设置 Fail2ban 可以阻止攻击者暴力破解您的密码库登录。如果您的实例是公开的，这一点尤其重要。

## [](#pre-requisite)预先说明

* 文件名位于每个代码块的顶部。

* 从 1.5.0 版开始，Vaultwarden 支持记录到文件。请设置[日志记录](/configuration/logging)。

* 尝试使用错误的账户信息登录到网页版密码库，并检查日志文件中如下格式的记录项：

复制

```
[YYYY-MM-DD hh:mm:ss][vaultwarden::api::identity][ERROR] Username or password is incorrect. Try again. IP: XXX.XXX.XXX.XXX. Username: email@domain.com.
```

## [](#installation)安装

### [](#debian-ubuntu-raspian-pi-os)Debian / Ubuntu / Raspian Pi OS

复制

```
sudo apt-get install fail2ban -y
```

### [](#fedora-centos)Fedora / Centos

需要 EPEL 库 (CentOS 7)

复制

```
sudo yum install epel-release
sudo yum install fail2ban -y
```

### [](#synology-dsm)群晖 DSM

使用 Synology 的话，由于各种原因需要做更多的工作。使用 Docker Compose 的完整的解决方案发布在[这里](https://github.com/sosandroid/docker-fail2ban-synology)。主要的问题是：

1. 嵌入式 IP 禁令系统不适用于 Docker 容器

2. 嵌入式 iptables 不支持 `REJECT` 块类型

3. Docker GUI 不允许某些高级设置

4. 修改系统配置不符合升级要求

因此，我们将在 Docker 容器中使用 Fail2ban。[Crazy-max/docker-fail2ban](https://github.com/crazy-max/docker-fail2ban) 提供了一个很好的解决方案，并且 Synology 的 Docker GUI 将被忽略。通过 SSH 的命令行，执行下列步骤（根据您的 Synology 配置调整 `volumeX`）：

1、获取 root 权限

复制

```
sudo -i
```

2、创建持久性文件夹

复制

```
mkdir -p /volumeX/docker/fail2ban/action.d/
mkdir -p /volumeX/docker/fail2ban/jail.d/
mkdir -p /volumeX/docker/fail2ban/filter.d/
```

3、将 blocktype 的 `REJECT` 替换为 `DROP` 块类型

复制

```
# /volumeX/docker/fail2ban/action.d/iptables.local

[Init]
blocktype = DROP
[Init?family=inet6]
blocktype = DROP
```

4、创建 docker-compose 文件

复制

```
# /volumeX/docker/fail2ban/docker-compose.yml

version: '3'
services:
	fail2ban:
		container_name: fail2ban
		restart: always
		image: crazymax/fail2ban:latest
		environment: 
		- TZ=Europe/Paris
		- F2B_DB_PURGE_AGE=30d
		- F2B_LOG_TARGET=/data/fail2ban.log
		- F2B_LOG_LEVEL=INFO
		- F2B_IPTABLES_CHAIN=INPUT

		volumes:
		- /volumeX/docker/fail2ban:/data
		- /volumeX/docker/vw-data:/vaultwarden:ro

		network_mode: "host"

		privileged: true
		cap_add:
			- NET_ADMIN
			- NET_RAW
```

5、使用命令行启动容器

复制

```
cd /volumeX/docker/fail2ban
docker-compose up -d
```

您现在应该看到该容器在 Synolog 的 Docker GUI 中运行了。在配置筛选器和 jail 后，您必须重新加载。

## [](#setup-for-web-vault)为网页密码库设置

按照惯例，`path_f2b` 代表 Fail2ban 工作所需的路径。这取决于您的系统，例如在 Synology 上是 `/volumeX/docker/fail2ban/`，但在其他系统上是 `/etc/fail2ban/`。

### [](#filter)Filter

使用如下内容创建文件：

复制

```
# path_f2b/filter.d/vaultwarden.local

[INCLUDES]
before = common.conf

[Definition]
failregex = ^.*?Username or password is incorrect\. Try again\. IP: <ADDR>\. Username:.*$
ignoreregex =
```

**提示**：如果在 `fail2ban.log` 中出现以下错误消息 (CentOS 7, Fail2Ban v0.9.7) `fail2ban.filter [5291]: ERROR No 'host' group in '^.*Username or password is incorrect\. Try again\. IP: <ADDR>\. Username:.*$'` 请将 `vaultwarden.local` 中的 `<ADDR>` 改为 `<HOST>`。

**提示**：对于 Cloudflare 用户，请确保在**管理面板** -> **高级设置** -> **客户端 IP 标头**中将客户端 IP 标头设置为 `CF-Connecting-IP`，否则客户端的真实 IP 将不会被识别和阻止。

**提示**：如果您在 `vaultwarden.log` 中看到 127.0.0.1 是登录失败的 IP 地址，那么您可能正在使用反向代理，而 Fail2ban 无法正常工作：

复制

```
[YYYY-MM-DD hh:mm:ss][vaultwarden::api::identity][ERROR] Username or password is incorrect. Try again. IP: 127.0.0.1. Username: email@example.com.
```

要解决这个问题，需要通过 X-Real-IP 头将真实的远程地址转发给 Vaultwarden。如何操作呢？根据你使用的代理服务器不同而不同。例如，在 Caddy 2.x 中，当您定义反向代理时，同时定义 `header_up X-Real-IP {remote_host}`。更多信息请参阅[代理示例](/deployment/proxy-examples)。

### [](#jail)Jail

> \[**译者注**]：[什么是 Jail](https://docs.freebsd.org/zh-cn/books/arch-handbook/jail/)

使用如下内容创建文件：

复制

```
# path_f2b/jail.d/vaultwarden.local

[vaultwarden]
enabled = true
port = 80,443,8081
filter = vaultwarden
banaction = %(banaction_allports)s
logpath = /path/to/vaultwarden.log
maxretry = 3
bantime = 14400
findtime = 14400
```

#### [](#note-for-docker-users)Docker 用户注意事项

Docker 使用 FORWARD 链而不是默认的 INPUT 链。如果接收请求的机器将他们直接映射到 Docker 容器，那么无论容器里有什么（反向代理、Vaultwarden 等），链都需要适当地设置。默认的 `action` 被设置为`action_`（它使用 `banaction`，其别名我们设置为 `banaction_allports`），`action_` 已经考虑了链的问题，因此，只需设置 `chain` 即可。参阅[这个类似的问题](https://forum.openwrt.org/t/resolved-fail2ban-and-iptables-ip-bans-not-blocked/90057)。

复制

```
chain = FORWARD
```

#### [](#note-for-synology-dsm-docker-users)Synology DSM Docker 用户注意事项

请将 `chain` 设置为 `DOCKER-USER`

复制

```
chain = DOCKER-USER
```

#### [](#note-for-docker-users-with-fail2ban-v1.1.1.dev1-and-possibly-newer)使用 Fail2Ban v1.1.1.dev1（以及可能更高版本）的 Docker 用户注意事项

在 Fail2Ban v1.1.1.dev1 中，Debian 的默认 `banactions` 从 iptables 变成了 nftables（参阅[此处](https://github.com/fail2ban/fail2ban/commit/d0d07285234871bad3dc0c359d0ec03365b6dddc)）。另一方面，Docker（至少是 25.0.3 版）仍在使用 iptables。因此，`banaction = %(banaction_allports)s` 无法阻止对 Docker 容器的请求。在这种情况下，使用：

复制

```
banaction = iptables
```

如果您使用 systemd 来管理 Vaultwarden，您可以为 Fail2ban 使用 systemd-journal：

复制

```
backend = systemd
filter = vaultwarden[journalmatch='_SYSTEMD_UNIT=your_vaultwarden.service']
```

使用它们来代替 `logpath = `和 `filter = `变量。

**后端注意事项**：如果您使用 `sudo apt install` 等方式安装 fail2ban，`/etc/fail2ban/jail.conf` 可能会使用 systemd 作为默认的后端。此默认配置项将导致无法监控 logpath 日志。

将 `backend = pyinotify` 或 `backend = inotify` 添加到 `vaultwarden.local` 配置中：

复制

```
# path_f2b/jail.d/vaultwarden.local

[vaultwarden]
enabled = true
backend = pyinotify
port = 80,443,8081
filter = vaultwarden
banaction = %(banaction_allports)s
logpath = /path/to/vaultwarden.log
maxretry = 3
bantime = 14400
findtime = 14400
```

重启 fail2ban 以使更改生效：

复制

```
sudo systemctl restart fail2ban
```

**Cloudflare 用户注意事项：**如果您使用 Cloudflare 代理，您需要将 Cloudflare 添加到您的操作列表中，如[这个指南](https://niksec.com/using-fail2ban-with-cloudflare/)中所示。

重新加载 Fail2ban 使更改生效：

复制

```
sudo systemctl reload fail2ban
```

请根据您自己的需要自由修改这些选项。

## [](#setup-for-admin-page)为管理页面设置

如果您通过设置 `ADMIN_TOKEN` 环境变量启用了管理控制台，则可以使用 Fail2ban 来阻止攻击者暴力破解您的管理令牌。该过程与网页密码库相同。

### [](#filter-1)Filter

使用如下内容创建文件：

复制

```
# path_f2b/filter.d/vaultwarden-admin.local

[INCLUDES]
before = common.conf

[Definition]
failregex = ^.*Invalid admin token\. IP: <ADDR>.*$
ignoreregex =
```

**提示**：如果在 `fail2ban.log` 中出现以下错误消息：`ERROR NOK: ("No 'host' group in '^.*Invalid admin token\\. IP: <ADDR>.*$'")`，请将 `vaultwarden-admin.local` 中的 `<ADDR>` 改为 `<HOST>`

### [](#jail-1)Jail

使用如下内容创建文件：

复制

```
# path_f2b/jail.d/vaultwarden-admin.local

[vaultwarden-admin]
enabled = true
port = 80,443
filter = vaultwarden-admin
banaction = %(banaction_allports)s
logpath = /path/to/vaultwarden.log
maxretry = 3
bantime = 14400
findtime = 14400
```

**注意**：Docker 使用 FORWARD 链而不是默认的 INPUT 链。因此，当使用 Docker 时，请使用下面的 `action` 行替换掉 `banaction` 行：

复制

```
action = iptables-allports[name=vaultwarden, chain=FORWARD]
```

如果您使用 systemd 来管理 Vaultwarden，您同样可以在这里为 Fail2ban 使用 systemd-journal：

复制

```
backend = systemd
filter = vaultwarden-admin[journalmatch='_SYSTEMD_UNIT=your_vaultwarden.service']
```

使用它们来代替 `logpath = `和 `filter = `变量。

**后端注意事项**：如果您使用 `sudo apt install` 等方式安装 fail2ban，`/etc/fai2ban/jail.conf` 可能会使用 systemd 作为默认的后端。此默认配置项将导致无法监控 logpath 日志。

将 `backend = pyinotify` 或 `backend = inotify` 添加到 `vaultwarden.local` 配置中：

复制

```
# path_f2b/jail.d/vaultwarden.local

[vaultwarden]
enabled = true
backend = pyinotify
port = 80,443,8081
filter = vaultwarden
banaction = %(banaction_allports)s
logpath = /path/to/vaultwarden.log
maxretry = 3
bantime = 14400
findtime = 14400
```

重启 fail2ban 以使更改生效：

复制

```
sudo systemctl restart fail2ban
```

**Cloudflare 用户请注意事项：**如果您使用 Cloudflare 代理，您需要将 Cloudflare 添加到您的操作列表中，如[本指南](https://niksec.com/using-fail2ban-with-cloudflare/)中所示。

重新加载 Fail2ban 使更改生效：

复制

```
sudo systemctl reload fail2ban
```

## [](#setup-for-totp)为 TOTP 代码设置

按照惯例，`path_f2b` 表示 Fail2ban 运行所需的路径。这取决于您的系统。例如，在 Synology 上，我们讨论的是 `/volumeX/docker/fail2ban/`，而在其他一些系统上，我们讨论的是 `/etc/fail2ban/`。

### [](#filter-2)Filter

使用如下内容创建文件：

复制

```
# path_f2b/filter.d/vaultwarden-totp.local
# Fail2Ban filter for Vaultwarden TOTP

[INCLUDES]
before = common.conf

[Definition]
failregex = ^.*\[ERROR\] Invalid TOTP code! Server time: (.*) UTC IP: <ADDR>$
ignoreregex =
```

日志示例：

复制

```
[YYYY-MM-DD hh:mm:ss][vaultwarden::api::core::two_factor::authenticator][ERROR] Invalid TOTP code! Server time: YYYY-MM-DD hh:mm:ss UTC IP: 1.2.3.4
```

### [](#jail-2)Jail

使用如下内容创建文件：

复制

```
# path_f2b/jail.d/vaultwarden-totp.local

[vaultwarden-totp]
enabled = true
port = 80,443
filter = vaultwarden-totp
banaction = iptables-multiport[name=vaultwarden-totp, port="80,443", protocol=tcp]
logpath = /path/to/vaultwarden.log
maxretry = 3
bantime = 14400
findtime = 14400
```

重启 fail2ban 以使更改生效：

复制

```
sudo systemctl restart fail2ban
```

请根据您自己的需要自由修改这些选项。

## [](#testing-fail-2-ban)测试 Fail2ban

现在，尝试使用任何电子邮件地址登录 Vaultwarden（不必是有效电子邮件，只需是电子邮件格式即可）。如果它可以正常工作，您的 IP 将被阻止。运行以下命令来取消阻止的 IP：

复制

```
# 使用 Docker
sudo docker exec -t fail2ban fail2ban-client set vaultwarden unbanip XX.XX.XX.XX
# 未使用 Docker
sudo fail2ban-client set vaultwarden unbanip XX.XX.XX.XX
```

如果 Fail2ban 无法正常运行，请检查 Vaultwarden 日志文件的路径是否正确。对于 Docker：如果指定的日志文件未生成和/或更新，请确保将 `EXTENDED_LOGGING` 变量设置为 `true`（默认值），并且确保日志文件的路径是 Docker 内部的路径（当您使用 `/vw-data/:/data/` 时，日志文件应位于容器外部的 `/data/...` 中）。

还要确认 Docker 容器的时区与主机的时区是否一致。通过将日志文件中显示的时间与主机操作系统的时间进行比较来进行检查。如果它们不一致，则有多种解决方法。一种是使用 `-e "TZ = <timezone>"` 选项启动 Docker 。可用的时区（比如 `-e TZ = "Australia/Melbourne"`）列表在[这里](https://en.wikipedia.org/wiki/List_of_tz_database_time_zones)查看。

如果您使用的是 podman 而不是 Docker，则无法通过 `-e "TZ = <timezone>"` 来设置时区。可以按照以下指南解决此问题（当使用 alpine 镜像时）：<https://wiki.alpinelinux.org/wiki/Setting_the_timezone>。

## [](#selinux-problems)SELinux 中的问题

当使用 SELinux 时，SELinux 可能会阻止 Fail2ban 读取日志。如果是这样，请运行此命令： `sudo tail /var/log/audit/audit.log`。您应该会看到如下类似内容（当然，实际的审核 ID (pid) 会因您的情况而不一样）：

复制

```
type=AVC msg=audit(1571777936.719:2193): avc:  denied  { search } for  pid=5853 comm="fail2ban-server" name="containers" dev="dm-0" ino=1144588 scontext=system_u:system_r:fail2ban_t:s0 tcontext=unconfined_u:object_r:container_var_lib_t:s0 tclass=dir permissive=0
```

您可以使用 `grep 'type=AVC msg=audit(1571777936.719:2193)' /var/log/audit/audit.log | audit2why` 来找出真正的原因。`audit2allow -a` 将为您提供有关如何创建模块并允许 Fail2ban 访问日志的具体说明。

按照这些步骤操作后就结束了！Fail2ban 现在应该可以正常工作了。


# 第三方ai配置
### **1. 确认 Vaultwarden 日志路径**
Vaultwarden 默认会将日志记录到以下位置：
- **日志文件路径**：`/path/to/vaultwarden/logs/identity.log`
  - 如果你使用 Docker 部署，日志可能在容器内或挂载到宿主机的某个目录中。
  - 例如：`/var/log/vaultwarden/identity.log`

确保你知道日志文件的具体位置，并确认日志中包含失败登录尝试的信息，通常类似于以下格式：
```
[2025-04-02 18:00:00][WARN] Failed login attempt. IP: 192.168.1.100
```

---

### **2. 创建 Fail2Ban 过滤器**
Fail2Ban 使用过滤器来解析日志文件并提取攻击者的 IP 地址。我们需要为 Vaultwarden 创建一个自定义过滤器。

#### (1) 创建过滤器文件
创建一个新的过滤器文件，例如 `/etc/fail2ban/filter.d/vaultwarden.conf`：

```bash
sudo nano /etc/fail2ban/filter.d/vaultwarden.conf
```

#### (2) 编辑过滤器内容
在文件中添加以下内容：

```ini
[INCLUDES]
before = common.conf

[Definition]
_daemon = vaultwarden
failregex = ^\[\d{4}-\d{2}-\d{2} \d{2}:\d{2}:\d{2}\]\[WARN\] Failed login attempt\. IP: <ADDR>$
ignoreregex =
```

- **`_daemon`**：指定服务名称，这里是 `vaultwarden`。
- **`failregex`**：
  - 匹配日志中的失败登录尝试。
  - `<ADDR>` 是 Fail2Ban 的占位符，用于提取 IP 地址。
- **`ignoreregex`**：忽略某些特定的日志行（这里为空，表示不忽略任何内容）。

#### (3) 测试过滤器
使用 `fail2ban-regex` 工具测试过滤器是否能正确匹配日志中的失败登录尝试：

```bash
sudo fail2ban-regex /path/to/vaultwarden/logs/identity.log /etc/fail2ban/filter.d/vaultwarden.conf
```

如果测试成功，你会看到类似以下输出：
```
Lines: 10 lines, 0 ignored, 5 matched, 5 missed
```
- **`matched`** 表示成功匹配的日志行数。
- **`missed`** 表示未匹配的日志行数。

---

### **3. 配置 Fail2Ban Jail**
Jail 是 Fail2Ban 的规则集，用于定义如何处理匹配到的事件。

#### (1) 编辑 Jail 配置文件
编辑 Fail2Ban 的主配置文件 `/etc/fail2ban/jail.local`：

```bash
sudo nano /etc/fail2ban/jail.local
```

#### (2) 添加 Vaultwarden Jail 配置
在文件末尾添加以下内容：

```ini
[vaultwarden]
enabled  = true
filter   = vaultwarden
logpath  = /path/to/vaultwarden/logs/identity.log
maxretry = 5
bantime  = 3600
findtime = 600
action   = iptables[name=Vaultwarden, port=http, protocol=tcp]
```

- **`enabled`**：启用该 Jail。
- **`filter`**：指定使用的过滤器（即我们刚刚创建的 `vaultwarden`）。
- **`logpath`**：指定 Vaultwarden 的日志文件路径。
- **`maxretry`**：允许的最大失败尝试次数（例如 5 次）。
- **`bantime`**：封禁时间（单位为秒，例如 3600 秒 = 1 小时）。
- **`findtime`**：检测时间窗口（单位为秒，例如 600 秒 = 10 分钟）。
- **`action`**：指定封禁动作，这里使用 `iptables` 来阻止 HTTP 请求。

#### (3) 重新加载 Fail2Ban
保存配置后，重新加载 Fail2Ban 以应用更改：

```bash
sudo systemctl reload fail2ban
```

---

### **4. 验证配置**
#### (1) 检查 Jail 状态
运行以下命令查看 Vaultwarden Jail 是否已启用：

```bash
sudo fail2ban-client status vaultwarden
```

输出示例：
```
Status for the jail: vaultwarden
|- Filter
|  |- Currently failed: 0
|  |- Total failed:     5
|  `- File list:        /path/to/vaultwarden/logs/identity.log
`- Actions
   |- Currently banned: 1
   |- Total banned:     1
   `- Banned IP list:   192.168.1.100
```

#### (2) 模拟攻击
你可以通过多次尝试错误的登录信息来模拟攻击，然后检查 Fail2Ban 是否正确封禁了你的 IP。

---

### **5. 可选优化**
#### (1) 使用 Docker 部署
如果你使用 Docker 部署 Vaultwarden，可以通过挂载日志文件的方式将日志暴露给宿主机。例如：

```yaml
volumes:
  - ./vaultwarden-logs:/data/logs
```

然后在 Fail2Ban 中指定挂载的日志路径。

#### (2) 增强安全性
- **限制 SSH 访问**：仅允许基于密钥的身份验证。
- **启用双因素认证（2FA）**：确保所有用户都启用了 2FA。
- **定期更新 Vaultwarden**：保持最新版本以修复已知漏洞。
