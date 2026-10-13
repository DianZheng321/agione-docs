# ClusterD 安装、升级与卸载

本指南从获取安装包开始，带你完成 ClusterD 安装、首次登录、部署自己的 AGIOne 集群，以及后续升级和卸载。

| 项目 | 说明 |
| --- | --- |
| 适用人员 | 交付工程师、现场安装人员、平台运维人员 |
| 当前交付范围 | Linux x86_64 安装包；以 Ubuntu 22.04 为安装示例 |
| 默认网页地址 | `http://<ClusterD 主机 IP>:18093/` |
| 安装入口 | 解压目录中的 `install.sh` |
| 网页操作 | 使用自己创建的普通账号 |

::: tip 先分清两台机器的用途
**ClusterD 主机**运行安装网页和任务服务；**AGIOne 目标节点**运行 Kubernetes、中间件和业务服务。只需在 ClusterD 主机安装一次 ClusterD，不需要在每台目标节点重复安装。

两者可以使用同一台机器，但必须一起规划磁盘、端口和资源。安装好 ClusterD 并不代表 AGIOne 已部署，后续还需要在网页中选择目标节点并执行安装。
:::

本指南适用于 `install.sh` + systemd 的交付包。旧包的 `start.sh` / `8080` 启动方式不适用。ARM、CentOS、macOS 不属于本版安装流程已核对的交付范围，不能直接照搬本指南。

## 操作时间线

| 阶段 | 你要做什么 | 完成标志 |
| --- | --- | --- |
| 1. 获取软件包 | 选择全量包或轻量包，取得对应 SHA-256 | 安装包版本、架构和校验值已确认 |
| 2. 准备主机 | 确认系统、基础工具、磁盘和网络 | 主机能运行 systemd，并能连接目标节点 |
| 3. 下载并校验 | 下载、核对外层压缩包、解压、校验包内文件 | 所有校验命令成功退出 |
| 4. 安装 ClusterD | 执行 `install.sh install` | 服务为 `active`，网页返回 HTTP 200 |
| 5. 首次使用 | 创建普通账号，填写自己的集群并预检 | 预检通过后开始 AGIOne 部署 |
| 6. 后续维护 | 按需升级或卸载 | 升级后数据保留；卸载前备份完成 |

## 术语速查

| 术语 | 通俗说明 |
| --- | --- |
| 全量包 | 自带 ClusterD 程序、执行引擎、Python 和离线部署资源，适合离线交付 |
| 轻量包 | 自带 ClusterD 程序和执行引擎，不含离线部署资源及内置 Python 安装资产 |
| SHA-256 | 文件的校验值，用于判断下载的文件是否与可信交付文件一致；不是数字签名 |
| systemd | Linux 服务管理工具，用于启动、停止和设置 ClusterD 开机启动 |
| Endpoint | 集群对外连接入口；访问 AGIOne 时以页面给出的 Web 访问地址为准 |
| `data_dir` | ClusterD 保存账号、集群、任务和资源记录的数据目录 |
| `secret.key` | 保护已保存敏感信息的加密密钥，必须与数据库一起备份 |

## 1. 获取安装包

**ClusterD 下载地址：待提供。**

::: warning 下载地址尚未发布
本页暂不提供可点击的下载链接。请向交付方取得本次版本的安装包下载地址及外层压缩包 SHA-256 校验值。下方命令中的 `<CLUSTERD_PACKAGE_URL>` 和 `XXX` 都是占位符，替换后才能执行，不要使用旧 ClusterD 下载地址代替。
:::

### 1.1 选择哪种包

| 安装包 | 文件名示例 | 适用场景与前提 |
| --- | --- | --- |
| 全量包，推荐离线交付使用 | `agione-clusterd-full-release-XXX-linux-x86_64.tar.gz` | 带离线部署资源；安装后可选择离线部署 AGIOne |
| 轻量包 | `agione-clusterd-release-XXX-linux-x86_64.tar.gz` | 主机需要 Python 3.10 或以上；后续部署需在线获取资源，或先导入已验证的分类离线资源 |

下面以全量包为例。使用轻量包时，将下载地址和文件名换成对应轻量包的值，安装、升级命令不变。

**确认三项信息再开始：安装包版本、Linux x86_64 架构、该压缩包的可信 SHA-256。**不要自己对刚下载的文件计算一个摘要，再把它当作交付方提供的校验值。

## 2. 安装前准备

### 2.1 主机与基础工具

所有命令在 **ClusterD 主机**的 Bash 终端执行。需要具备系统 `root` 或 `sudo` 执行能力；这是主机安装权限，与网页登录账号无关。

| 检查项 | 要求 |
| --- | --- |
| 操作系统 | 本指南使用 Ubuntu 22.04、x86_64，并具备正常运行的 systemd |
| 基础工具 | Bash、系统 `python3`、`sha256sum`、`tar`、`gzip`、coreutils、OpenSSH 客户端；联网下载还需要 `curl` 和 CA 证书 |
| 磁盘空间 | 同时考虑压缩包、解压内容、安装后的资源、临时工作区和升级备份；不可只按压缩包大小预留 |
| 目标节点访问 | ClusterD 主机可以通过私网直连或已配置的跳板机 SSH 访问目标节点，目标节点已启动 SSH 服务 |
| 网页访问 | 在防火墙或安全组放行 TCP `18093`，仅允许可信运维网络访问 |
| Docker | 安装 ClusterD 本身不要求预装 Docker |

全量包里的 Python 是首次包校验**之后**才安装的，不能代替首次校验所需的系统 `python3`。完全离线主机必须提前备好基础工具。

联网 Ubuntu 主机缺少工具时，可先执行：

```bash
sudo apt-get update
sudo apt-get install -y python3 openssh-client ca-certificates curl tar gzip coreutils
```

### 2.2 先检查，不要重复安装

```bash
uname -m
python3 --version
df -h /opt /var/lib /var/backups
sudo systemctl --no-pager --full status agione-clusterd
```

`uname -m` 应输出 `x86_64`。新主机提示找不到 `agione-clusterd` unit 是正常的；如果已经存在 ClusterD，请转到 [6. 升级 ClusterD](#_6-升级-clusterd)，不要按全新安装处理。

## 3. 下载、校验与解压

### 3.1 下载到 ClusterD 主机

先进入 root Bash，后续命令沿用同一个终端。`/opt/clusterd-packages` 仅用于保存交付包，不是 ClusterD 的运行目录。

```bash
sudo -i
bash
set -e
mkdir -p /opt/clusterd-packages
cd /opt/clusterd-packages
```

将下面的 URL、文件名和 SHA-256 替换为本次实际交付值。SHA-256 是 64 位十六进制文本，不是下载地址。

```bash
CLUSTERD_PACKAGE_URL="<CLUSTERD_PACKAGE_URL>"
CLUSTERD_PACKAGE_NAME="agione-clusterd-full-release-XXX-linux-x86_64.tar.gz"
CLUSTERD_PACKAGE_SHA256="<TRUSTED_OUTER_ARCHIVE_SHA256>"

curl --fail --location --output "$CLUSTERD_PACKAGE_NAME" "$CLUSTERD_PACKAGE_URL"
```

完全离线时，可通过交付介质或 `scp` 将安装包放到同一目录，再设置实际文件名和可信 SHA-256；不需要执行 `curl`。

### 3.2 核对外层压缩包

```bash
printf '%s  %s\n' "$CLUSTERD_PACKAGE_SHA256" "$CLUSTERD_PACKAGE_NAME" | sha256sum -c -
```

输出 `<安装包文件名>: OK` 才能继续。出现 `FAILED`、摘要格式错误或找不到文件时，停止操作，确认版本和校验值，必要时重新下载安装包。SHA-256 只能核对内容，校验值本身必须来自可信交付渠道。

### 3.3 解压并校验包内文件

```bash
CLUSTERD_PACKAGE_DIR="${CLUSTERD_PACKAGE_NAME%.tar.gz}"
tar -xzf "$CLUSTERD_PACKAGE_NAME"
cd "/opt/clusterd-packages/$CLUSTERD_PACKAGE_DIR"
bash verify.sh
```

`verify.sh` 检查包内 manifest、`SHA256SUMS` 和资源；**只有整个命令成功退出**才表示通过。包内校验不能代替解压前的外层压缩包校验。

::: warning 不要修改解压目录
不要把自己的配置、安装日志或上传文件放进解压目录，也不要改动其中的文件。包校验会检查文件集合，新增或修改文件可能导致安装失败。日志写入 `/var/log`，安装后的实际配置位于 `/etc/agione-clusterd`。
:::

## 4. 安装 ClusterD

### 4.1 执行安装

继续在上一步的解压目录、root Bash 中执行：

```bash
CLUSTERD_INSTALL_LOG=/var/log/agione-clusterd-install.log
if bash install.sh install > "$CLUSTERD_INSTALL_LOG" 2>&1; then
  tail -n 20 "$CLUSTERD_INSTALL_LOG"
else
  tail -n 100 "$CLUSTERD_INSTALL_LOG"
  false
fi
```

脚本会校验安装包、安装程序和运行资源、生成配置与 systemd 单元，并启动服务。成功结束时显示 `agione-clusterd-install-ok`；失败时先查看日志，不要继续执行首次登录流程。

全量包会读取多 GB 资源，安装和校验可能持续较长时间。没有连续输出不代表卡死；不要重复启动安装，不要在执行期间移动或删除解压目录。

只想先检查包和安装路径时，可以执行 `bash install.sh install --dry-run`。输出 `release-dry-run-ok` 只表示检查通过，**不代表已安装服务或 AGIOne**。

### 4.2 检查安装结果

```bash
systemctl is-active agione-clusterd
systemctl --no-pager --full status agione-clusterd
curl --fail --silent --show-error --output /dev/null \
  --write-out 'HTTP %{http_code}\n' http://127.0.0.1:18093/
```

正常应看到 `active` 和 `HTTP 200`。然后在自己的电脑浏览器打开：

```text
http://<ClusterD 主机 IP>:18093/
```

本机访问正常但浏览器打不开时，先检查 `18093` 端口的防火墙、安全组和路由，不要直接关闭整个防火墙。

### 4.3 需要了解的目录

| 默认路径 | 用途 |
| --- | --- |
| `/opt/agione-clusterd` | ClusterD 程序、执行引擎和固定安装资源 |
| `/opt/agione-python` | 全量包安装的 Python 运行环境 |
| `/etc/agione-clusterd/config.yaml` | 监听地址、数据目录和资源策略等配置 |
| `/etc/agione-clusterd/env` | 服务环境变量，可能含敏感信息，不要公开 |
| `/var/lib/agione-clusterd` | 数据库、密钥、集群和任务记录、资源缓存 |
| `/var/log/agione-clusterd-install.log` | 本次安装日志 |

如需更换网页端口，编辑 `/etc/agione-clusterd/config.yaml` 的 `addr`，再执行 `systemctl restart agione-clusterd`。如果 `env` 中设置了 `AGIONE_CLUSTERD_ADDR`，该值会覆盖 YAML。同步更新防火墙、安全组和浏览器地址。

## 5. 使用普通账号部署自己的集群

### 5.1 首次创建账号

1. 在可信运维网络打开 ClusterD 网页。全新数据目录首次访问会显示“创建账户”。
2. 按页面填写自己的用户名和密码，密码至少 8 位，使用独立强密码。
3. 创建成功后进入页面。后续访问使用已有账号登录，不需要每次重新注册。

如果页面已经显示登录页或旧集群，先确认该主机是否沿用了已有数据目录；不要为了重新注册而删除数据库。

ClusterD 网页账号、目标节点的 SSH 账号、AGIOne 业务登录账号是三套不同的账号，不要混用。

### 5.2 从操作向导开始

普通账号在“我的集群”中查看和操作自己创建的集群。当前每个普通账号在同一项目下可创建一个未退役的 AGIOne 集群。

1. 打开“操作向导”或在“我的集群”选择“创建我的集群”。
2. 选择安装模式。全量包离线部署时，确认“本地资源已就绪”；轻量包需先确认在线资源可用，或导入匹配的离线资源。
3. 填写集群名称、服务选择和目标节点信息，包括节点 IP、SSH 连接地址、端口、用户名及认证信息。没有公网的节点需要可达私网或已配置的跳板机，不会自动穿透网络。
4. 确认存储和网络规划。需要 HTTPS 时准备与访问域名匹配的有效证书，仅开放 `443` 不等于完成 HTTPS 配置。
5. 执行预检。先处理所有阻断项，并阅读警告，再开始安装；不要绕过磁盘不足、连接失败或旧环境残留。
6. 查看任务日志，等待部署任务结束。在“我的集群”检查节点、服务和健康状态。
7. 打开详情页给出的 **Web 访问地址**，领取并安全保存页面提供的集群登录凭据，验证 AGIOne 登录和实际业务访问。

::: tip 在线与离线资源
新交付包的在线资源读取使用预置公开 HTTPS 配置，客户无需填写云存储 AK/SK。在线部署前仍需确认页面所需 Release 全部就绪，且目标维护节点能访问资源地址；在线资源不可用时，不要把“ClusterD 服务已启动”当成资源已就绪。

全量包选择离线安装时，不需要再次下载同一版本的内置资源。额外导入分类离线资源时，必须同时提供对应的 manifest 和 bundle。
:::

**安装完成的判断：**不仅是任务显示成功，还要确认节点和服务就绪、Web 可访问、业务账号能登录。ClusterD 首页 HTTP 200 只证明 ClusterD 自身可访问。

## 6. 升级 ClusterD

::: warning 升级的是 ClusterD，不是正在运行的 AGIOne
升级会更新 ClusterD 程序、网页和执行引擎，并重启 ClusterD。全量包还带有资源快照，但不会自动更新已运行集群的 AGIOne 业务服务。
:::

### 6.1 升级前准备与备份

1. 按第 1～3 节取得并校验新版本安装包，解压到新的版本目录，不覆盖旧解压目录。
2. 在维护窗口操作，等待所有部署、预检、更新等任务结束，不允许仍有 `queued` 或 `running` 任务。
3. 确认实际安装路径和数据目录。下面是默认路径示例；自定义配置、`data_dir` 或安装路径必须按实际值调整。

```bash
sudo systemctl cat agione-clusterd
sudo awk '/^data_dir:/ {print}' /etc/agione-clusterd/config.yaml
df -h /var/backups
```

进入 root Bash，停止服务及清理计划后备份，避免只复制正在使用的 SQLite `.db` 文件而遗漏 WAL 数据：

```bash
sudo -i
bash
set -e
if systemctl cat agione-clusterd-housekeeping.timer >/dev/null 2>&1; then
  systemctl stop agione-clusterd-housekeeping.timer
fi
if systemctl cat agione-clusterd-housekeeping.service >/dev/null 2>&1; then
  systemctl stop agione-clusterd-housekeeping.service
fi
systemctl stop agione-clusterd

CLUSTERD_BACKUP_ROOT="/var/backups/agione-clusterd-$(date +%Y%m%dT%H%M%S)"
install -d -m 0700 "$CLUSTERD_BACKUP_ROOT"
cp -a /etc/agione-clusterd "$CLUSTERD_BACKUP_ROOT/config" && \
cp -a /var/lib/agione-clusterd "$CLUSTERD_BACKUP_ROOT/data"
test -s "$CLUSTERD_BACKUP_ROOT/data/agione-clusterd.db"
test -s "$CLUSTERD_BACKUP_ROOT/data/secret.key"
printf 'Backup: %s\n' "$CLUSTERD_BACKUP_ROOT"
```

如旧版本没有 housekeeping 单元，仅该单元的“不存在”可以跳过；必须确认 ClusterD 和实际清理进程已停止，复制命令成功结束。数据库与 `secret.key` 必须成对备份。备份包含敏感信息，应限制访问并安全保存；缓存可能很大，先预留足够空间，不要将备份放在待清理目录中。

`set -e` 让失败命令中断当前 Bash，避免备份失败后继续升级。若暂缓升级，确认原配置和数据未被修改后，重新启动 `agione-clusterd` 及原先启用的 housekeeping timer，恢复正常运行。

### 6.2 执行升级

在**新版本**解压目录中，以 root Bash 执行。`XXX` 替换为已校验的新版本；轻量包使用对应轻量包目录。自定义安装路径需沿用原来的安装参数。

```bash
cd /opt/clusterd-packages/agione-clusterd-full-release-XXX-linux-x86_64
bash install.sh upgrade --dry-run
```

检查通过后执行：

```bash
CLUSTERD_UPGRADE_LOG=/var/log/agione-clusterd-upgrade.log
if bash install.sh upgrade > "$CLUSTERD_UPGRADE_LOG" 2>&1; then
  tail -n 20 "$CLUSTERD_UPGRADE_LOG"
else
  tail -n 100 "$CLUSTERD_UPGRADE_LOG"
  false
fi
```

成功应显示 `agione-clusterd-upgrade-ok`。升级保留已有配置、账号、数据库、密钥和客户状态；新的服务目录默认值可能以 `.upgrade.new` 文件提供，需人工比较，不要直接覆盖已有配置。

### 6.3 升级后检查

执行第 4.2 节的服务与 HTTP 检查，使用原账号登录，确认自己的集群和任务记录仍在。默认安装路径还可以核对新包与已安装程序：

```bash
sha256sum bin/agione-clusterd /opt/agione-clusterd/bin/agione-clusterd
CLUSTERD_RUNNING_PID="$(systemctl show agione-clusterd --property MainPID --value)"
test "$CLUSTERD_RUNNING_PID" -gt 0 && \
sha256sum "/proc/$CLUSTERD_RUNNING_PID/exe"
```

三处程序摘要应一致，避免出现“安装文件换了，但进程仍运行旧程序”。升级失败时保留日志和备份，先判断数据库兼容性再制定恢复方案；不要把安装器保留的 previous 文件当作一键完整回滚。

## 7. 卸载 ClusterD

当前 `install.sh` 只提供 `install` 和 `upgrade`，**没有 `uninstall` 子命令**。需要卸载时按本节处理，不要尝试用安装或升级命令清空数据。

### 7.1 先选择你要卸载什么

| 你的目的 | 应采取的操作 |
| --- | --- |
| 暂停网页和任务服务 | `sudo systemctl stop agione-clusterd`，不删除程序和数据 |
| 移除 ClusterD 主机上的服务 | 按本节备份、停止和删除 ClusterD 自身文件 |
| 同时移除 AGIOne 目标节点环境 | 先在网页对自己的集群完成对应远端环境清理，确认成功后再卸载 ClusterD |

::: danger 卸载边界
卸载 ClusterD 不会自动停止或清理远端 AGIOne、Kubernetes 和持久化数据。“仅删除 ClusterD 记录”同样不会卸载远端服务。

删除 `/var/lib/agione-clusterd` 会同时删除账号、集群记录、任务证据、资源缓存和 `secret.key`。没有完成备份或尚有活动任务时，不要执行下方删除命令。
:::

### 7.2 备份并确认路径

先按第 6.1 节核对实际路径，停止服务并完成一致性备份。确认待删除目录不是挂载点或软链接，也没有被其他服务使用。

下面仅适用于**全部使用默认路径**的安装。尤其要确认 `/opt/agione-python` 和 `/etc/kubemore` 没有被其他部署工具或服务复用；如有共享，只处理 ClusterD 独占文件，不直接删除共享目录。自定义安装不能直接复制下方命令。

### 7.3 移除服务单元和自身文件

继续在 root Bash 执行。housekeeping 单元仅在存在时处理；其他失败应查明原因后继续。

```bash
set -e
if systemctl cat agione-clusterd-housekeeping.timer >/dev/null 2>&1; then
  systemctl disable --now agione-clusterd-housekeeping.timer
fi
if systemctl cat agione-clusterd-housekeeping.service >/dev/null 2>&1; then
  systemctl stop agione-clusterd-housekeeping.service
fi
systemctl disable --now agione-clusterd

rm -f /etc/systemd/system/agione-clusterd.service \
  /etc/systemd/system/agione-clusterd-housekeeping.service \
  /etc/systemd/system/agione-clusterd-housekeeping.timer
systemctl daemon-reload
```

再次确认备份可读、服务已停止、路径为本机默认安装目录且未被共享，才执行永久删除：

```bash
rm -rf /opt/agione-clusterd /etc/agione-clusterd /var/lib/agione-clusterd
rm -rf /opt/agione-python /etc/kubemore
```

保留安装日志和备份，以便审计或恢复。不要删除 `/var/lib/agione`、`/var/lib/rancher/k3s`、`/etc/rancher/k3s` 或 Docker/containerd 数据目录，也不要为了卸载 ClusterD 清空全局 systemd journal。

### 7.4 验证卸载结果

```bash
systemctl list-unit-files | grep -F agione-clusterd || true
test ! -e /opt/agione-clusterd
test ! -e /etc/agione-clusterd
test ! -e /var/lib/agione-clusterd
ss -ltnp | grep ':18093' || true
```

正常结果：ClusterD 服务单元已移除，独占安装目录不存在，`18093` 不再由 ClusterD 监听。远端 AGIOne 是否卸载，必须以远端清理任务与节点实际状态为准，不能根据 ClusterD 网页打不开判断。

重新安装时使用新的可信安装包，明确是否恢复旧数据。恢复时数据库与 `secret.key` 必须配对，不要把旧密钥与空数据库混用，也不要未经审查恢复旧配置中的资源地址。

## 8. 常见问题

| 现象 | 先检查什么 |
| --- | --- |
| `python3 is required` | 主机是否具备首次包校验所需的系统 Python |
| 外层或包内校验失败 | 安装包版本、可信 SHA-256、是否有缺失或被修改的文件；重新获取可信包，不修改校验清单 |
| 安装无连续输出 | 查看独立日志、进程和磁盘；不要重复启动同一安装 |
| 服务未启动 | `sudo journalctl -u agione-clusterd -n 100 --no-pager`，检查配置、路径和端口占用 |
| 本机 HTTP 200，远端浏览器打不开 | 监听地址、`18093` 防火墙、安全组和路由 |
| 在线安装不可用 | 页面所需资源版本是否就绪，以及 ClusterD / 目标维护节点的 DNS、HTTPS 与出口网络 |
| 目标节点预检失败 | 在“我的集群”打开最近任务日志，先处理 SSH、资源、端口或旧环境残留等具体阻断项 |
| 升级后数据看起来丢失 | 核对实际 `data_dir` 和环境变量，不要先删除数据库或创建新密钥 |

排障时记录版本、任务编号、失败阶段和脱敏日志，不要公开 `env`、SSH 密码、集群登录凭据或 `secret.key`。
