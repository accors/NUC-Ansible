# agent-nuc-ansible 共享契约

> 本文件是所有 role 与项目级文件之间的唯一接口定义。附件实际版本为 **1.4**；任务说明中的旧章节号按语义映射到 v1.4。任何 role 不得自创同义变量、handler 或目录所有权。

## 1. 适用范围与事实来源

- 目标系统：全新 Debian 13.6 (`trixie`, `amd64`) 上的 ASUS NUC 13 Pro。
- 第一阶段只部署 Debian 基础防护、SSH、Docker、Codex、Tailscale、Paseo、cloudflared、受限 agent-runner 与 Restic。
- 同一事项发生冲突时，优先级为：本契约的跨 role 接口 > PDF v1.4 的对应章节 > role 内部实现细节。
- v1.4 章节映射：`base` 见 6.1、6.3、6.4、7.2、7.3、12.4、13.2；`ssh_harden` 见 7.1；`srv_layout` 见 9.1；`docker` 见 9.2、9.3；`codex` 见 8.1、11.3；`tailscale` 见 8.4、10.1、10.2；`paseo` 见 8.4-8.6；`cloudflared` 见 10.3-10.5；`agent_runner` 见 11.1-11.3；`restic` 见 12.1-12.4。

## 2. 变量命名表

所有非秘密变量使用 `nuc_` 前缀；秘密使用 `vault_nuc_` 前缀。下表列出允许写入 `group_vars/all/main.yml` 的完整变量集合。role 的 `defaults/main.yml` 只能重述自己使用的这些变量，默认值必须一致。

其中六项是环境相关值：`nuc_admin_user`、`nuc_admin_authorized_keys`、`nuc_lan_cidr`、`nuc_tailscale_ipv4`、`nuc_access_hostname`、`nuc_restic_repository`。它们不写入提交进仓库的 `main.yml`，改由不提交的 `group_vars/all/local.yml` 提供（模板 `local.example.yml` 只含占位符），因此下表中它们的「默认值」列记为 `local.yml`，`main.yml` 与 role 的 `defaults/main.yml` 都不为其设默认值。变量名、类型与语义不受影响。

### 2.1 主机、账号与网络

| 变量 | 类型 | 默认值 | 来源 | 说明 |
|---|---|---|---|---|
| `nuc_admin_user` | string | `local.yml` | 5、7.1、8.3 | 已由 Debian 安装器创建的个人管理员；Paseo 与交互式 Codex 使用此账号 |
| `nuc_admin_authorized_keys` | list[string] | `local.yml` | 7.1 | 至少一把公钥；为空时 `ssh_harden` 必须在关闭密码认证前失败 |
| `nuc_lan_cidr` | string | `local.yml` | 7.2 | 用 `ip route` 实测；只允许该网段访问 SSH |
| `nuc_tailscale_ipv4` | string | `local.yml` | 8.5、10.2 | 人工授权后用 `tailscale ip -4` 实测；必须属于 `100.64.0.0/10`。**不进入 `daemon.listen`，但必须进入 `daemon.hostnames`**，否则 tailnet 客户端会被 Host 校验拒绝 |
| `nuc_access_hostname` | string | `local.yml` | 8.5、10.4 | Cloudflare Published application 域名；必须进入 Paseo `daemon.hostnames` |
| `nuc_paseo_port` | integer | `6767` | 7.3、8.5 | Paseo 监听端口；`daemon.listen` 固定为 `0.0.0.0:<port>`，LAN 侧隔离由 UFW 负责 |
| `nuc_agent_runner_user` | string | `agent-runner` | 11.1 | 无 sudo、无 Docker 组的自动化账号 |
| `nuc_agent_runner_home` | path | `/var/lib/agent-runner` | 11.1、11.3 | agent-runner 的系统 home |
| `nuc_agent_runner_workdir` | path | `/srv/automation` | 9.1、11.1、11.3 | 自动化工作目录；必须与 `/srv/workspaces` 平级 |
| `nuc_agent_runner_codex_home` | path | `/var/lib/agent-runner/codex` | 11.1、11.3 | systemd unit 的 `CODEX_HOME` |
| `nuc_timezone` | string | `Australia/Melbourne` | 5、13.2 | timer 与主机使用的时区 |
| `nuc_debian_codename` | string | `trixie` | 4、9.2 | 第三方 APT 仓库发行代号 |
| `nuc_architecture` | string | `amd64` | 4、9.2 | APT 仓库架构 |

### 2.2 目录、基础防护与 SSH

| 变量 | 类型 | 默认值摘要 | 来源 | 说明 |
|---|---|---|---|---|
| `nuc_srv_directories` | list[dict] | 见 3.1 | 9.1、11.1 | 每项固定含 `path`、`owner`、`group`、`mode`、`purpose`、`managed_by` |
| `nuc_base_packages` | list[string] | `openssh-server`、`curl`、`ca-certificates`、`gnupg`、`git`、`tmux`、`ripgrep`、`jq`、`ufw`、`unattended-upgrades`、`nvme-cli`、`smartmontools`、`lm-sensors`、`btop`、`restic` | 6.1 | Debian 基础包集合 |
| `nuc_sleep_targets` | list[string] | 四个 sleep target | 6.3 | `sleep.target`、`suspend.target`、`hibernate.target`、`hybrid-sleep.target` |
| `nuc_unattended_upgrades_auto_reboot` | boolean | `false` | 6.4 | 第一阶段必须保持关闭 |
| `nuc_journald_system_max_use` | string | `1G` | 任务补充、12.4 | journald 磁盘上限；PDF 未给数值，采用保守默认值 |
| `nuc_nvme_device` | path | `/dev/nvme0` | 12.4 | smartctl/nvme-cli 监控目标 |
| `nuc_ufw_lan_ssh_port` | integer | `22` | 7.2 | LAN SSH 端口 |
| `nuc_ufw_tailscale_ports` | list[integer] | `[22, 6767]` | 7.2、7.3 | `tailscale0` 显式允许端口；第二项必须等于 `nuc_paseo_port` |
| —（无变量） | — | — | 7.2、7.3 | 另有一条 `deny in from nuc_lan_cidr to nuc_paseo_port/tcp`，复用上述两个变量，不新增契约变量 |
| `nuc_sshd_hardening_options` | dict | 见 PDF 7.1 | 7.1 | 禁 root、启公钥、禁密码/KbdInteractive/X11 |

### 2.3 软件版本与服务配置

PDF v1.4 只明确固定 Debian 13.6 与 Node.js 22 LTS。其余软件没有给出可信的精确构建号，因此默认值表示 PDF 指定的稳定发行通道；若部署方需要可复现构建，可把空字符串或 `latest` 改成仓库中真实存在的精确版本，role 必须尊重该值，不能在代码里另写版本。

| 变量 | 类型 | 默认值 | 来源 | 说明 |
|---|---|---|---|---|
| `nuc_docker_package_versions` | dict[string,string] | 五个 Docker 包均为 `""` | 9.2 | 空值表示 Docker stable 当前候选；非空时按 `包=版本` 安装 |
| `nuc_docker_log_max_size` | string | `10m` | 9.3 | 合并进现有 `daemon.json` |
| `nuc_docker_log_max_file` | string | `3` | 9.3 | Docker 要求字符串值 |
| `nuc_codex_installer_url` | string | `https://chatgpt.com/codex/install.sh` | 8.1 | 管理员账户的官方安装脚本 |
| `nuc_codex_admin_version` | string | `latest` | 8.1 | 官方脚本通道 pin；安装后必须记录实际版本与路径 |
| `nuc_codex_system_version` | string | `latest` | 11.3 | `/usr/local` 下 `@openai/codex` 的 npm 版本 pin |
| `nuc_tailscale_channel` | string | `stable` | 8.4、10.2 | Tailscale APT 通道 |
| `nuc_tailscale_package_version` | string | `""` | 8.4 | 空值表示 stable 当前候选 |
| `nuc_nodejs_major` | integer | `22` | 8.4 | NodeSource LTS 主版本 pin |
| `nuc_nodejs_package_version` | string | `""` | 8.4 | 空值表示 NodeSource 22.x 当前候选 |
| `nuc_paseo_cli_version` | string | `latest` | 8.4、8.9 | `@getpaseo/cli` npm 版本 pin |
| `nuc_paseo_npm_prefix_relative` | path | `.local/npm` | 8.4 | 相对管理员 home 的 npm prefix |
| `nuc_paseo_password_configured` | boolean | `false` | 8.5、8.6 | 人工执行 `set-password` 后改为 `true`，才允许启用 user unit |
| `nuc_cloudflared_package_version` | string | `""` | 10.3 | 空值表示 Cloudflare 仓库当前候选 |
| `nuc_cloudflared_enable_service` | boolean | `true` | 10.4 | Dashboard 已建 tunnel 且 vault token 已填后安装/启用 service |
| `nuc_restic_package_version` | string | `""` | 6.1、12.2 | 空值表示 Debian 13.6 当前候选 |

### 2.4 agent-runner 与 Restic

| 变量 | 类型 | 默认值 | 来源 | 说明 |
|---|---|---|---|---|
| `nuc_agent_runner_credential_path` | path | `/etc/credstore.encrypted/agent-codex-api-key` | 11.3 | Ansible 只 `stat`，绝不创建凭据 |
| `nuc_agent_runner_task_prompt` | string | `执行每日自动化任务，并把结果写入 reports/` | 11.3 | `codex exec` 的明确任务 |
| `nuc_agent_runner_schedule` | string | `*-*-* 07:30:00 Australia/Melbourne` | 11.3 | systemd timer `OnCalendar` |
| `nuc_agent_runner_randomized_delay` | string | `10m` | 11.3 | timer 随机延迟 |
| `nuc_agent_runner_timer_enabled` | boolean | `false` | 11.3 | 人工首次运行成功后改为 `true` |
| `nuc_restic_repository` | string | `local.yml` | 12.2 | 仓库路径或不含嵌入凭据的远端 URL；会暴露备份位置，不写入 main.yml |
| `nuc_restic_backup_paths` | list[path] | `/srv`、`/etc/ssh`、`/etc/systemd/system` | 12.1、12.2 | 每晚备份范围 |
| `nuc_restic_excludes` | list[string] | 见下 | 8.2、12.2 | 排除缓存、worktree、依赖与 Codex 登录态 |
| `nuc_restic_keep` | dict | `{daily: 7, weekly: 4, monthly: 6, yearly: 1}` | 12.2、12.3 | `forget --prune` 保留策略 |
| `nuc_restic_schedule` | string | `*-*-* 02:30:00 Australia/Melbourne` | 12.3 | nightly timer |
| `nuc_restic_randomized_delay` | string | `30m` | 12.3 | timer 随机延迟 |
| `nuc_restic_read_data_subset` | string | `5%` | 12.2、12.4 | 人工/月度完整性检查读取比例 |
| `nuc_restic_env_file` | path | `/etc/restic/agent-nuc.env` | 12.2、12.3 | root-only 环境文件 |
| `nuc_restic_wrapper_path` | path | `/usr/local/sbin/agent-restic-backup` | 任务补充、12.2 | 串行执行 backup 与 forget 的包装脚本 |
| `nuc_restic_initialize_repository` | boolean | `true` | 任务补充、12.2 | `snapshots` 探测失败且确认为未初始化时执行 `restic init` |
| `nuc_restic_enable_timer` | boolean | `true` | 任务补充、12.3 | 安装并启用 nightly service/timer |

`nuc_restic_excludes` 的固定默认值：

```yaml
nuc_restic_excludes:
  - "**/node_modules"
  - "/srv/worktrees"
  - "**/.cache"
  - "**/.codex/auth.json"
```

### 2.5 Vault 变量

`group_vars/all/vault.example.yml` 只提交占位符；真实的 `vault.yml` 必须经 `ansible-vault` 加密并由 `.gitignore` 排除。同样地，`inventory.yml` 与 `files/preseed.cfg` 只提交 `.example` 模板，真实文件由 `.gitignore` 排除。

| 变量 | 用途 | 来源 |
|---|---|---|
| `vault_nuc_restic_password` | Restic 独立随机长密码 | 12.2 |
| `vault_nuc_cloudflared_tunnel_token` | Dashboard 创建 tunnel 后得到的 service token | 10.4 |

下列内容不进入 Ansible Vault：管理员 `~/.codex/auth.json`、Paseo 内层密码、个人 SSH 私钥、真实公钥指纹、agent-runner 的 OpenAI API key。最后一项只允许通过 `systemd-creds` 人工写入 `nuc_agent_runner_credential_path`。

### 2.6 运行时 fact（不得写入 group_vars）

| fact | 产生 role | 消费 role | 规则 |
|---|---|---|---|
| `nuc_admin_uid` | `paseo` | `paseo` | 从 `getent passwd` 探测，禁止硬编码 1000 |
| `nuc_admin_home` | `codex`，`paseo` 可重探测 | `codex`、`paseo` | 从 passwd 数据得到 |
| `nuc_codex_admin_bin` | `codex` | `paseo` | 安装后用管理员登录环境的 `command -v codex` 探测 |
| `nuc_codex_admin_bin_dir` | `codex` | `paseo` | `nuc_codex_admin_bin` 的目录 |
| `nuc_tailscale_detected_ipv4` | `tailscale` | `paseo` | `tailscale ip -4` 的只读结果；必须等于配置值 |
| `nuc_paseo_unit_path` | `paseo` | `paseo` | 必须含 `nuc_codex_admin_bin_dir`、Paseo npm bin 与系统路径 |

## 3. 目录与权限矩阵

### 3.1 `/srv` 清单

`nuc_srv_directories` 必须精确表达下表；每个路径只有一个 owner role。

| path | owner:group | mode | 用途 | owner role |
|---|---|---|---|---|
| `/srv/workspaces` | `{{ nuc_admin_user }}:{{ nuc_admin_user }}` | `0750` | 长期项目与 Agent 工作区 | `srv_layout` |
| `/srv/worktrees` | `{{ nuc_admin_user }}:{{ nuc_admin_user }}` | `0750` | 可清理重建的 Git worktree | `srv_layout` |
| `/srv/stacks` | `{{ nuc_admin_user }}:{{ nuc_admin_user }}` | `0750` | Compose 定义、环境模板、服务文档 | `srv_layout` |
| `/srv/data` | `{{ nuc_admin_user }}:{{ nuc_admin_user }}` | `0750` | 服务持久化数据与备份重点 | `srv_layout` |
| `/srv/automation` | `{{ nuc_agent_runner_user }}:{{ nuc_agent_runner_user }}` | `0750` | 受限自动化工作区 | `agent_runner` |

`/srv/automation` 必须与 `/srv/workspaces` 平级，绝不能放在后者之下；`srv_layout` 不得预创建它。

### 3.2 其他受控路径

| path | owner:group | mode | 用途 | owner role |
|---|---|---|---|---|
| 管理员 `~/.ssh` | 管理员 | `0700` | SSH 公钥目录 | `ssh_harden` |
| 管理员 `~/.ssh/authorized_keys` | 管理员 | `0600` | 登录公钥 | `ssh_harden` |
| 管理员 `~/.local/npm` | 管理员 | `0755` | Paseo npm prefix | `paseo` |
| 管理员 `~/.paseo` | 管理员 | `0700` | Paseo 配置与状态 | `paseo` |
| 管理员 `~/.paseo/config.json` | 管理员 | `0600` | Paseo 唯一配置来源 | `paseo` |
| 管理员 `~/.config/systemd/user` | 管理员 | `0755` | user unit 目录 | `paseo` |
| `/var/lib/agent-runner` | agent-runner | `0750` | 受限账号 home | `agent_runner` |
| `/var/lib/agent-runner/codex` | agent-runner | `0750` | `CODEX_HOME` | `agent_runner` |
| `/etc/credstore.encrypted` | `root:root` | `0700` | systemd 加密凭据目录；只创建目录 | `agent_runner` |
| `/etc/restic` | `root:root` | `0700` | Restic 配置目录 | `restic` |
| `/etc/restic/agent-nuc.env` | `root:root` | `0600` | 仓库地址与密码环境文件 | `restic` |
| `/usr/local/sbin/agent-restic-backup` | `root:root` | `0750` | nightly 包装脚本 | `restic` |

## 4. Handler 命名表

全局名称区分 role，subagent 只能 `notify` 下列名称；不需要延迟执行的服务操作直接写成幂等 task。

| handler 名称 | 定义 role | 触发条件 |
|---|---|---|
| `base | restart unattended-upgrades` | `base` | 自动更新配置变化 |
| `base | restart smartd` | `base` | smartd 配置变化 |
| `base | restart systemd-journald` | `base` | journald 上限变化 |
| `ssh_harden | reload ssh` | `ssh_harden` | 已通过 `sshd -t` 的加固配置变化 |
| `docker | restart docker` | `docker` | 合并后的 `daemon.json` 变化 |
| `tailscale | restart tailscaled` | `tailscale` | tailscaled 配置变化 |
| `paseo | daemon-reload user units` | `paseo` | Paseo user unit 变化 |
| `paseo | restart paseo` | `paseo` | 已启用服务的 config/unit 变化 |
| `cloudflared | restart cloudflared` | `cloudflared` | cloudflared service 配置变化 |
| `agent_runner | daemon-reload system units` | `agent_runner` | automation service/timer 变化 |
| `restic | daemon-reload system units` | `restic` | backup service/timer 变化 |

## 5. Tag 与执行顺序

role tag 与目录名完全相同，`site.yml` 只按下列顺序表达依赖，不使用 `meta/dependencies`：

| 顺序 | role/tag | 主要前置 |
|---:|---|---|
| 1 | `base` | 裸 Debian 已能用管理员 + sudo 连接 |
| 2 | `ssh_harden` | `nuc_admin_authorized_keys` 非空且另一个会话已验证密钥 |
| 3 | `srv_layout` | 管理员账号存在 |
| 4 | `docker` | 基础网络与 ca-certificates 可用 |
| 5 | `codex` | 管理员账号存在；安装后人工 `codex login` |
| 6 | `tailscale` | 安装后人工 `sudo tailscale up`，再重跑完成地址校验 |
| 7 | `paseo` | Codex 已登录、Tailscale 地址已验证；config 写入后人工设密码 |
| 8 | `cloudflared` | Dashboard 已建 tunnel/Access，token 已写入 vault |
| 9 | `agent_runner` | Node/npm 可用；人工 systemd-creds 与首次 service 验收 |
| 10 | `restic` | 仓库地址和 vault 密码已填写 |

不得创建第二套细分 tag。交互门禁通过布尔变量与清单表达，避免 `paseo:install` 一类带冒号 tag 在不同调用方式中的歧义。

## 6. 风格与实现约束

- 目标 `ansible-core >= 2.16`；模块一律使用 FQCN，如 `ansible.builtin.*`、`community.general.*`、`ansible.posix.*`。
- task、handler、模板注释使用简体中文；命令、路径、systemd 字段等技术字面量保持原文。
- 每个 role 必须包含 `tasks/main.yml`、`defaults/main.yml`、`meta/main.yml`；按需添加 `handlers/`、`templates/`、`files/`。
- `meta/main.yml` 的 `dependencies` 必须为空；跨 role 依赖只由 `site.yml` 顺序表达。
- 所有 task 必须幂等；第二次 `--check --diff` 应无变更。读取状态的 `command` 必须 `changed_when: false`；会写状态的 `command` 必须有可靠的 `creates`/`removes` 或明确的状态探测与 `changed_when`。
- 优先使用模块，禁止用 `shell`/`command` 代替 `apt`、`file`、`template`、`service`、`user`、`authorized_key` 等已有模块。
- 所有下载使用 TLS、固定官方来源，并通过 `get_url`/APT keyring 管理；不得使用 `curl | sh`。若 PDF 只展示安装脚本，role 应展开为可审计的下载与条件执行。
- 秘密不得进入 task 名、日志、模板 diff 或命令输出；涉及 vault token/password 的 task 必须 `no_log: true`。不得提交真实密码、token、密钥、公钥指纹。
- `docker` 必须读取并递归合并现有 `/etc/docker/daemon.json`，保留契约外已有键；不得整体覆盖。
- `ssh_harden` 必须先落地 `authorized_keys`，再写 `10-hardening.conf`；写配置的任务必须带 `validate: /usr/sbin/sshd -t -f %s`。
- UFW 必须含 LAN SSH 规则、`tailscale0` 上的 22/6767，以及一条显式拒绝 `nuc_lan_cidr` 访问 `nuc_paseo_port` 的规则；接口未出现不影响先写规则。
- Paseo `config.json` 是唯一配置来源；unit 的 `ExecStart` 除 `daemon start --foreground` 外不得传配置参数。`daemon.listen` 固定 `0.0.0.0:<port>`，不绑定 `tailscale0` 地址——绑定会让 daemon 依赖 tailscaled 存活，把 Cloudflare Access 那条路径一并拖下水；unit 中也不得再加等待 tailscale 接口的 `ExecStartPre`。
- Paseo user unit PATH 必须含 `nuc_codex_admin_bin_dir` 的真实探测值；不得照抄固定 PATH。`loginctl enable-linger` 必须先于任何 `systemctl --user`，后者必须显式传 `XDG_RUNTIME_DIR=/run/user/{{ nuc_admin_uid }}`。
- agent-runner 的 Codex 必须安装为 `/usr/local/bin/codex`；`ProtectHome=yes` 下 `HOME`/`CODEX_HOME` 指向 `/var/lib/agent-runner`，`ReadWritePaths` 只含 `/srv/automation` 与 `/var/lib/agent-runner`。
- agent-runner 不得加入 `sudo`、`docker` 或其他特权组，不得获得 Docker socket。

## 7. 禁止自动化与失败契约

下列步骤只能由人执行，必须写入 `docs/INTERACTIVE-CHECKLIST.md` 并标明位于哪个 tag 之前/之后。禁止 `expect`、`pexpect`、模拟按键或其他变通包装。

| 人工步骤 | 流程位置 | role 的允许行为 |
|---|---|---|
| BIOS 刷写及 BIOS 设置 | Ansible 之前（PDF 3、13.1） | 不探测、不代劳；清单记录验收项 |
| 管理员 `codex login` | `codex` 安装后、`paseo` 前（8.1、8.5） | 只运行 `codex login status` 读取状态；未登录时 `fail` 并给命令 |
| `sudo tailscale up` | `tailscale` 安装后、`paseo` 前（8.4、8.5、10.2） | 只运行 `tailscale ip -4`；无 100.x 或与配置不符时 `fail` |
| `paseo daemon set-password` | `config.json` 写入后、启用 user unit 前（8.5、8.6） | 以 `nuc_paseo_password_configured` 为人工确认门禁；为 `false` 时 `fail` |
| Cloudflare Dashboard 域名、Create tunnel、Published application、Access application/policy/MFA | `cloudflared` 安装 service 前（10.4、10.5） | 不调用 Dashboard/API；只消费 vault token 安装本机 service |
| `sudo systemd-creds encrypt - /etc/credstore.encrypted/agent-codex-api-key` | `agent_runner` 建目录/账号后、启用 automation 前（11.3） | 只 `stat` 凭据文件；不存在时 `fail` 并给命令 |

此外，automation timer 首次启用前必须由人手工启动一次 service、检查 journal，并把 `nuc_agent_runner_timer_enabled` 改为 `true`。Restic 密码必须另存离线副本；该备份动作也只写入清单。

## 8. Subagent 写入边界

- 一个 subagent 只拥有一个 `roles/<name>/` 目录，不得修改其他 role 或任何项目级文件。
- subagent 可创建自己 role 内的 task/default/meta/handler/template/file；不得写 `site.yml`、`README.md`、`docs/INTERACTIVE-CHECKLIST.md`、`CONVENTIONS.md`、inventory 或 group_vars。
- 发现契约不足时必须回报主 agent，不得自行扩展公共变量或 handler。
