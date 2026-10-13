# ClusterD Installation, Upgrade, and Uninstallation

This guide takes you from obtaining a release package to installing ClusterD, signing in for the first time, deploying your own AGIOne cluster, and later upgrading or uninstalling ClusterD.

| Item | Description |
| --- | --- |
| Intended readers | Delivery engineers, field installers, and platform operations staff |
| Current delivery scope | Linux x86_64 release packages; Ubuntu 22.04 is used in the examples |
| Default Web URL | `http://<ClusterD-host-IP>:18093/` |
| Installation entry | `install.sh` in the extracted package directory |
| Web operations | Use the regular account you create for yourself |

::: tip Distinguish the two machine roles
The **ClusterD host** runs the installation Web interface and task service. **AGIOne target nodes** run Kubernetes, middleware, and business services. Install ClusterD once on the ClusterD host, not on every target node.

The roles can share a machine, but plan their disk, ports, and resources together. Installing ClusterD does not deploy AGIOne; select target nodes and start deployment in the Web interface afterward.
:::

This guide applies to release packages using `install.sh` and systemd. The older `start.sh` / `8080` workflow does not apply. ARM, CentOS, and macOS are outside the verified delivery scope of this guide; do not reuse these instructions unchanged on those platforms.

## Workflow Timeline

| Stage | What you do | Completion signal |
| --- | --- | --- |
| 1. Obtain the package | Choose a full or light package and obtain its SHA-256 | Release version, architecture, and trusted checksum are confirmed |
| 2. Prepare the host | Check the system, tools, disk, and network | systemd works and target nodes are reachable |
| 3. Download and verify | Download, verify the outer archive, extract, and verify package contents | All verification commands exit successfully |
| 4. Install ClusterD | Run `install.sh install` | Service is `active` and the home page returns HTTP 200 |
| 5. First use | Create a regular account and precheck your cluster | Start AGIOne deployment after precheck passes |
| 6. Maintain | Upgrade or uninstall when needed | Upgrade preserves data; backup precedes uninstallation |

## Terminology Quick Reference

| Term | Plain explanation |
| --- | --- |
| Full package | Includes the ClusterD program, execution engine, Python, and offline deployment resources |
| Light package | Includes the program and execution engine, but not offline deployment resources or embedded Python installation assets |
| SHA-256 | A file checksum that compares downloaded contents with a trusted delivery; it is not a digital signature |
| systemd | Linux service management for starting, stopping, and enabling ClusterD at boot |
| Endpoint | A cluster connection entry; use the Web access URL displayed on the page to visit AGIOne |
| `data_dir` | Directory holding ClusterD accounts, clusters, tasks, and resource records |
| `secret.key` | Encryption key protecting stored sensitive information; back it up with the database |

## 1. Obtain the Release Package

**ClusterD download URL: to be provided.**

::: warning The download URL has not been published
No clickable download link is available on this page yet. Obtain the package URL and the outer archive SHA-256 from your delivery contact. `<CLUSTERD_PACKAGE_URL>` and `XXX` below are placeholders; replace them before running the commands. Do not substitute an older ClusterD package URL.
:::

### 1.1 Choose a package

| Package | Example filename | Use case and prerequisites |
| --- | --- | --- |
| Full package, recommended for offline delivery | `agione-clusterd-full-release-XXX-linux-x86_64.tar.gz` | Includes offline resources; select offline AGIOne deployment after installation |
| Light package | `agione-clusterd-release-XXX-linux-x86_64.tar.gz` | Host needs Python 3.10 or later; deployments need online resources or previously imported, verified categorized offline resources |

The following steps use a full package. For a light package, substitute its URL and filename; installation and upgrade commands remain the same.

**Confirm the release version, Linux x86_64 architecture, and trusted archive SHA-256 before starting.** Computing a checksum from the file you just downloaded does not replace the checksum supplied through the delivery channel.

## 2. Prepare the Host

### 2.1 Host and basic tools

Run all commands in Bash on the **ClusterD host**. System `root` or `sudo` execution is required for installation; it is separate from the Web login account.

| Check | Requirement |
| --- | --- |
| Operating system | These examples use Ubuntu 22.04, x86_64, with functioning systemd |
| Basic tools | Bash, system `python3`, `sha256sum`, `tar`, `gzip`, coreutils, and an OpenSSH client; online downloads also need `curl` and CA certificates |
| Disk space | Allow for the archive, extracted package, installed resources, temporary workspace, and upgrade backups; archive size alone is insufficient |
| Target access | The ClusterD host can SSH to target nodes over a private network or configured jump host; target SSH services are running |
| Web access | Allow TCP `18093` through the firewall or security group only from a trusted operations network |
| Docker | Docker is not required to install ClusterD itself |

The embedded Python in a full package is installed **after** initial package verification. It cannot replace system `python3` for that first check. Prepare basic tools in advance on an air-gapped host.

On a connected Ubuntu host, install missing tools when needed:

```bash
sudo apt-get update
sudo apt-get install -y python3 openssh-client ca-certificates curl tar gzip coreutils
```

### 2.2 Check before installing again

```bash
uname -m
python3 --version
df -h /opt /var/lib /var/backups
sudo systemctl --no-pager --full status agione-clusterd
```

`uname -m` should print `x86_64`. A missing `agione-clusterd` unit is expected on a new host. If ClusterD already exists, follow [6. Upgrade ClusterD](#_6-upgrade-clusterd) instead of treating the host as a new installation.

## 3. Download, Verify, and Extract

### 3.1 Download to the ClusterD host

Enter a root Bash shell and keep using the same terminal. `/opt/clusterd-packages` stores delivery packages; it is not the ClusterD runtime directory.

```bash
sudo -i
bash
set -e
mkdir -p /opt/clusterd-packages
cd /opt/clusterd-packages
```

Replace the URL, filename, and SHA-256 below with the actual delivery values. SHA-256 is a 64-character hexadecimal string, not a download URL.

```bash
CLUSTERD_PACKAGE_URL="<CLUSTERD_PACKAGE_URL>"
CLUSTERD_PACKAGE_NAME="agione-clusterd-full-release-XXX-linux-x86_64.tar.gz"
CLUSTERD_PACKAGE_SHA256="<TRUSTED_OUTER_ARCHIVE_SHA256>"

curl --fail --location --output "$CLUSTERD_PACKAGE_NAME" "$CLUSTERD_PACKAGE_URL"
```

For an air-gapped host, transfer the package to this directory using delivery media or `scp`, then set the actual filename and trusted SHA-256. Skip `curl`.

### 3.2 Verify the outer archive

```bash
printf '%s  %s\n' "$CLUSTERD_PACKAGE_SHA256" "$CLUSTERD_PACKAGE_NAME" | sha256sum -c -
```

Continue only when the output is `<package-filename>: OK`. If it reports `FAILED`, an invalid checksum format, or a missing file, stop and confirm the version and checksum. Download the package again if necessary. SHA-256 verifies contents; the checksum itself must come from a trusted delivery channel.

### 3.3 Extract and verify package contents

```bash
CLUSTERD_PACKAGE_DIR="${CLUSTERD_PACKAGE_NAME%.tar.gz}"
tar -xzf "$CLUSTERD_PACKAGE_NAME"
cd "/opt/clusterd-packages/$CLUSTERD_PACKAGE_DIR"
bash verify.sh
```

`verify.sh` checks the package manifests, `SHA256SUMS`, and resources. **The entire command must exit successfully.** This check does not replace outer archive verification before extraction.

::: warning Keep the extracted directory unchanged
Do not add personal configuration, installation logs, or uploaded files to the extracted directory, or modify its contents. Verification checks the file set, and added or modified files can block installation. Write logs to `/var/log`; installed configuration belongs in `/etc/agione-clusterd`.
:::

## 4. Install ClusterD

### 4.1 Run installation

Continue in the extracted directory from the previous step, in root Bash:

```bash
CLUSTERD_INSTALL_LOG=/var/log/agione-clusterd-install.log
if bash install.sh install > "$CLUSTERD_INSTALL_LOG" 2>&1; then
  tail -n 20 "$CLUSTERD_INSTALL_LOG"
else
  tail -n 100 "$CLUSTERD_INSTALL_LOG"
  false
fi
```

The script verifies the package, installs the program and runtime resources, generates configuration and systemd units, and starts the service. Successful completion prints `agione-clusterd-install-ok`. On failure, inspect the log before proceeding to first login.

A full package reads multiple GB of resources. Installation and verification can take time without continuous output. Do not start another installation or move or delete the extracted directory while the command is running.

To check the package and planned paths first, run `bash install.sh install --dry-run`. `release-dry-run-ok` means the check passed, **not that ClusterD or AGIOne is installed**.

### 4.2 Check the result

```bash
systemctl is-active agione-clusterd
systemctl --no-pager --full status agione-clusterd
curl --fail --silent --show-error --output /dev/null \
  --write-out 'HTTP %{http_code}\n' http://127.0.0.1:18093/
```

Expected results are `active` and `HTTP 200`. Then open the following URL in your computer's browser:

```text
http://<ClusterD-host-IP>:18093/
```

If local access works but the browser cannot connect, check the firewall, security group, and routing for `18093`. Do not disable the entire firewall.

### 4.3 Key directories

| Default path | Purpose |
| --- | --- |
| `/opt/agione-clusterd` | Program, execution engine, and fixed installation resources |
| `/opt/agione-python` | Python runtime installed from the full package |
| `/etc/agione-clusterd/config.yaml` | Listen address, data directory, and resource policies |
| `/etc/agione-clusterd/env` | Service environment variables; may contain sensitive information |
| `/var/lib/agione-clusterd` | Database, key, cluster and task records, and resource cache |
| `/var/log/agione-clusterd-install.log` | Installation log |

To change the Web port, edit `addr` in `/etc/agione-clusterd/config.yaml`, then run `systemctl restart agione-clusterd`. `AGIONE_CLUSTERD_ADDR`, if set in `env`, overrides YAML. Update the firewall, security group, and browser URL as well.

## 5. Deploy Your Cluster with a Regular Account

### 5.1 Create your first account

1. Open ClusterD from a trusted operations network. First access with a new data directory shows the account creation page.
2. Enter your own username and password. Use a unique, strong password of at least 8 characters.
3. After account creation, you enter the interface. Use that account to sign in on later visits; registration is not repeated after each restart.

If the login page or existing clusters appear immediately, check whether the host reuses an existing data directory. Do not delete the database to reopen registration.

The ClusterD Web account, target-node SSH accounts, and AGIOne business login accounts are separate. Do not use them interchangeably.

### 5.2 Start from the operation wizard

A regular account uses My Cluster to view and operate the cluster it created. Currently, one non-retired AGIOne cluster can be created per account in the same project.

1. Open the operation wizard, or choose Create My Cluster in My Cluster.
2. Select the installation mode. For offline deployment from a full package, confirm that local resources are ready. For a light package, first confirm online availability or import matching offline resources.
3. Enter the cluster name, service selection, and target nodes, including node IPs, SSH connection addresses, ports, usernames, and authentication details. Nodes without public access need reachable private networking or a configured jump host; connectivity is not established automatically.
4. Confirm storage and network planning. HTTPS requires a valid certificate matching the access domain; allowing `443` alone does not configure HTTPS.
5. Run precheck. Resolve every blocker and review warnings before installation. Do not bypass insufficient disk space, connection failures, or existing-environment residue.
6. Open task logs and wait for deployment to finish. Check nodes, services, and health in My Cluster.
7. Open the **Web access URL** shown in the details, retrieve and securely store the cluster login credentials offered on the page, and verify AGIOne login and actual business access.

::: tip Online and offline resources
New delivery packages use preconfigured public HTTPS access for online resources; customers do not need cloud storage AK/SK. Before online deployment, confirm that all required releases on the page are ready and that the target maintenance node can reach their URLs. A running ClusterD service does not prove online resources are ready.

Offline installation from a full package does not require downloading the same bundled resources again. Additional categorized offline imports must include both the matching manifest and bundle.
:::

**Completion criteria:** a successful task is not enough. Confirm node and service readiness, Web access, and business login. HTTP 200 from the ClusterD home page proves only that ClusterD itself is reachable.

## 6. Upgrade ClusterD

::: warning This upgrades ClusterD, not the running AGIOne deployment
An upgrade updates the ClusterD program, Web interface, and execution engine, and restarts ClusterD. A full package also includes a resource snapshot, but does not automatically update business services in running clusters.
:::

### 6.1 Prepare and back up

1. Obtain and verify the new package using sections 1-3. Extract into a new version directory without overwriting the old extracted directory.
2. Use a maintenance window. Wait for deployment, precheck, update, and other tasks to finish; no tasks may remain `queued` or `running`.
3. Confirm actual installation and data paths. The following examples use defaults; adjust custom configuration, `data_dir`, and installation paths to match the host.

```bash
sudo systemctl cat agione-clusterd
sudo awk '/^data_dir:/ {print}' /etc/agione-clusterd/config.yaml
df -h /var/backups
```

Enter root Bash, stop the service and housekeeping, then back up. Copying only an active SQLite `.db` file can omit WAL data:

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

If an older installation has no housekeeping unit, only that unit's absence may be skipped. Confirm that ClusterD and any actual housekeeping process are stopped and that both copy commands succeed. Back up the database and `secret.key` together. Backups contain sensitive information; restrict access and store them securely. Caches can be large, so reserve sufficient space and keep backups outside directories slated for removal.

`set -e` stops the current Bash shell on a failed command, preventing upgrade after a failed backup. If you postpone the upgrade, confirm that the original configuration and data are unchanged, then restart `agione-clusterd` and any previously enabled housekeeping timer to restore normal operation.

### 6.2 Run upgrade

Use root Bash in the **new version's** extracted directory. Replace `XXX` with the verified new version; use the matching directory for a light package. Keep the original installation arguments for custom paths.

```bash
cd /opt/clusterd-packages/agione-clusterd-full-release-XXX-linux-x86_64
bash install.sh upgrade --dry-run
```

After the check passes, run:

```bash
CLUSTERD_UPGRADE_LOG=/var/log/agione-clusterd-upgrade.log
if bash install.sh upgrade > "$CLUSTERD_UPGRADE_LOG" 2>&1; then
  tail -n 20 "$CLUSTERD_UPGRADE_LOG"
else
  tail -n 100 "$CLUSTERD_UPGRADE_LOG"
  false
fi
```

Successful completion prints `agione-clusterd-upgrade-ok`. Upgrade preserves configuration, accounts, database, key, and customer state. New service catalog defaults may be provided in `.upgrade.new` files for comparison; do not overwrite existing configuration blindly.

### 6.3 Verify the upgrade

Repeat the service and HTTP checks in section 4.2. Sign in with your existing account and confirm that your cluster and task records remain. For default installation paths, also compare the new package with the installed program:

```bash
sha256sum bin/agione-clusterd /opt/agione-clusterd/bin/agione-clusterd
CLUSTERD_RUNNING_PID="$(systemctl show agione-clusterd --property MainPID --value)"
test "$CLUSTERD_RUNNING_PID" -gt 0 && \
sha256sum "/proc/$CLUSTERD_RUNNING_PID/exe"
```

All three binary hashes should match. This prevents mistaking a replaced file for an updated running process. On failure, retain logs and backups and assess database compatibility before planning recovery. The installer's previous files are not a complete one-click rollback.

## 7. Uninstall ClusterD

The current `install.sh` provides only `install` and `upgrade`; **there is no `uninstall` subcommand**. Follow this section for removal instead of using an installation or upgrade command to erase data.

### 7.1 Decide what to remove

| Goal | Action |
| --- | --- |
| Pause the Web and task service | Run `sudo systemctl stop agione-clusterd`; keep program and data |
| Remove the service from the ClusterD host | Back up, stop, and remove ClusterD-owned files as described below |
| Also remove AGIOne target-node environments | First complete the corresponding remote cleanup for your cluster in the Web interface, confirm success, then uninstall ClusterD |

::: danger Uninstallation boundaries
Uninstalling ClusterD does not automatically stop or clean up remote AGIOne, Kubernetes, or persistent data. Deleting only ClusterD records does not uninstall remote services either.

Deleting `/var/lib/agione-clusterd` removes accounts, cluster records, task evidence, resource caches, and `secret.key`. Do not run deletion commands before completing backup or while tasks are active.
:::

### 7.2 Back up and confirm paths

Use section 6.1 to check actual paths, stop services, and take a consistent backup. Confirm that directories to be removed are not mount points or symbolic links and are not used by other services.

The following commands apply **only to an installation using all default paths**. In particular, confirm that `/opt/agione-python` and `/etc/kubemore` are not shared with other deployment tools or services. If shared, remove only ClusterD-exclusive files, not the shared directories. Do not copy these commands unchanged for a custom installation.

### 7.3 Remove service units and owned files

Continue in root Bash. Handle housekeeping units only when present; investigate other failures before proceeding.

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

Confirm again that the backup is readable, services are stopped, and these are unshared default installation paths before permanent deletion:

```bash
rm -rf /opt/agione-clusterd /etc/agione-clusterd /var/lib/agione-clusterd
rm -rf /opt/agione-python /etc/kubemore
```

Keep installation logs and backups for audit or recovery. Do not delete `/var/lib/agione`, `/var/lib/rancher/k3s`, `/etc/rancher/k3s`, or Docker/containerd data directories. Do not clear the global systemd journal to uninstall ClusterD.

### 7.4 Verify removal

```bash
systemctl list-unit-files | grep -F agione-clusterd || true
test ! -e /opt/agione-clusterd
test ! -e /etc/agione-clusterd
test ! -e /var/lib/agione-clusterd
ss -ltnp | grep ':18093' || true
```

Expected results: ClusterD units are removed, exclusive installation directories are absent, and ClusterD no longer listens on `18093`. Determine remote AGIOne removal from cleanup-task results and actual node state, not from an unreachable ClusterD page.

For reinstallation, use a new trusted package and explicitly decide whether to restore old data. Restore the database and `secret.key` as a pair; do not combine an old key with an empty database or restore old resource URLs without reviewing the configuration.

## 8. Troubleshooting

| Symptom | First check |
| --- | --- |
| `python3 is required` | System Python needed for initial package verification |
| Outer or internal verification fails | Package version, trusted SHA-256, and missing or modified files; obtain a trusted package again instead of editing checksum manifests |
| No continuous installation output | Independent log, process state, and disk; do not start a duplicate installation |
| Service does not start | `sudo journalctl -u agione-clusterd -n 100 --no-pager`; check configuration, paths, and port conflicts |
| Local HTTP 200 but browser cannot connect | Listen address, `18093` firewall rules, security groups, and routing |
| Online installation is unavailable | Required resource releases and DNS, HTTPS, and outbound access from ClusterD / the target maintenance node |
| Target-node precheck fails | Open the latest task log in My Cluster and resolve specific SSH, resource, port, or old-environment blockers |
| Data appears missing after upgrade | Actual `data_dir` and environment overrides; do not delete the database or generate a replacement key first |

Record the version, task ID, failure stage, and redacted logs when troubleshooting. Do not publish `env`, SSH passwords, cluster login credentials, or `secret.key`.
