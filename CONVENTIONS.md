# agent-nuc-ansible 共享契约

> 本文件是所有 role 与项目级文件之间的唯一接口定义。附件实际版本为 **1.4**；任务说明中的旧章节号按语义映射到 v1.4。任何 role 不得自创同义变量、handler 或目录所有权。

## 1. 适用范围与事实来源

- 目标系统：全新 Debian 13.6 (`trixie`, `amd64`) 上的 ASUS NUC 13 Pro。
- 第一阶段只部署 Debian 基础防护、SSH、Docker、Codex、Tailscale、Paseo、cloudflared、受限 agent-runner、只读 ops 账号 `solar` 与 Restic。
- 同一事项发生冲突时，优先级为：本契约的跨 role 接口 > PDF v1.4 的对应章节 > role 内部实现细节。
- v1.4 章节映射：`base` 见 6.1、6.3、6.4、7.2、7.3、12.4、13.2；`ssh_harden` 见 7.1；`srv_layout` 见 9.1；`docker` 见 9.2、9.3；`codex` 见 8.1、11.3；`tailscale` 见 8.4、10.1、10.2；`paseo` 见 8.4-8.6；`cloudflared` 见 10.3-10.5；`agent_runner` 见 11.1-11.3；`ops_agent` 来自任务补充的观察者设计；`restic` 见 12.1-12.4。

## 2. 变量命名表

所有非秘密变量使用 `nuc_` 前缀；秘密使用 `vault_nuc_` 前缀。下表列出允许写入 `group_vars/all/00-contract.yml` 的完整变量集合。role 的 `defaults/main.yml` 只能重述自己使用的这些变量，默认值必须一致。

其中十项是环境相关值：`nuc_hostname`、`nuc_admin_user`、`nuc_admin_authorized_keys`、`nuc_lan_cidr`、`nuc_static_ipv4`、`nuc_lan_gateway`、`nuc_lan_dns`、`nuc_tailscale_ipv4`、`nuc_access_hostname`、`nuc_restic_repository`。它们不写入提交进仓库的 `00-contract.yml`，改由不提交的 `group_vars/all/local.yml` 提供（模板 `local.yml.example` 只含占位符），因此下表中它们的「默认值」列记为 `local.yml`，`00-contract.yml` 与 role 的 `defaults/main.yml` 都不为其设默认值。变量名、类型与语义不受影响。

### 2.1 主机、账号与网络

| 变量 | 类型 | 默认值 | 来源 | 说明 |
|---|---|---|---|---|
| `nuc_hostname` | string | `local.yml` | 任务补充 | 目标机系统主机名；`base` 写入 `/etc/hostname` 与 `/etc/hosts` 的 `127.0.1.1`，设置前必须校验为合法 RFC 1123 标签 |
| `nuc_admin_user` | string | `local.yml` | 5、7.1、8.3 | 已由 Debian 安装器创建的个人管理员；Paseo 与交互式 Codex 使用此账号 |
| `nuc_admin_authorized_keys` | list[string] | `local.yml` | 7.1 | 至少一把公钥；为空时 `ssh_harden` 必须在关闭密码认证前失败 |
| `nuc_lan_cidr` | string | `local.yml` | 7.2 | 用 `ip route` 实测；只允许该网段访问 SSH |
| `nuc_static_ipv4` | string | `local.yml` | 任务补充 | enp86s0 的静态地址；掩码由 `nuc_lan_cidr` 前缀推导，不单独设变量。必须落在 `nuc_lan_cidr` 内、非网络/广播地址、且不等于网关，`network` 写入 profile 前逐条断言 |
| `nuc_lan_gateway` | string | `local.yml` | 任务补充 | 默认网关；实测 `ip route` 的 default via |
| `nuc_lan_dns` | list[string] | `local.yml` | 任务补充 | DNS 解析器，至少一个。切换静态后 DHCP 不再下发 DNS，必须显式给出 |
| `nuc_lan_interface` | string | `enp86s0` | 任务补充 | LAN 物理接口；systemd PCI 路径命名，换主板或加 PCIe 网卡才会变 |
| `nuc_lan_connection_name` | string | `Wired connection 1` | 任务补充 | 要修改的 NetworkManager profile 名；`network` 只改实测已激活的这一个，名字不符即失败，**不得新建 profile** |
| `nuc_lan_route_metric` | integer | `100` | 任务补充 | 以太网默认路由 metric；必须小于 WiFi 的 metric（实测 `wlo1` 为 600），否则 WiFi 抢走默认路由 |
| `nuc_tailscale_ipv4` | string | `local.yml` | 8.5、10.2 | 人工授权后用 `tailscale ip -4` 实测；必须属于 `100.64.0.0/10`。**不进入 `daemon.listen`，但必须进入 `daemon.hostnames`**，否则 tailnet 客户端会被 Host 校验拒绝 |
| `nuc_access_hostname` | string | `local.yml` | 8.5、10.4 | Cloudflare Published application 域名；必须进入 Paseo `daemon.hostnames`，因此**必须在运行 `paseo` 之前填好**，`paseo` 第一条 task 拦占位值与非法域名 |
| `nuc_paseo_port` | integer | `6767` | 7.3、8.5 | Paseo 监听端口；`daemon.listen` 固定为 `0.0.0.0:<port>`，LAN 侧隔离由 UFW 负责 |
| `nuc_agent_runner_user` | string | `agent-runner` | 11.1 | 无 sudo、无 Docker 组的自动化账号 |
| `nuc_agent_runner_home` | path | `/var/lib/agent-runner` | 11.1、11.3 | agent-runner 的系统 home |
| `nuc_agent_runner_workdir` | path | `/srv/automation` | 9.1、11.1、11.3 | 自动化工作目录；必须与 `/srv/workspaces` 平级 |
| `nuc_agent_runner_codex_home` | path | `/var/lib/agent-runner/codex` | 11.1、11.3 | systemd unit 的独立 `CODEX_HOME` 与 OAuth 登录缓存目录 |
| `nuc_ops_agent_user` | string | `solar` | 任务补充 | 仅加入 `systemd-journal`，不得加入 `sudo`、`docker`、`disk` |
| `nuc_ops_agent_home` | path | `/home/solar` | 任务补充 | 独立账号 home，必须为 `0700` |
| `nuc_ops_agent_workspace` | path | `/var/lib/openclaw-ops-agent/workspace` | 任务补充 | root 控制启动指令；只给 memory/reports 运行时写权限 |
| `nuc_ops_agent_gateway_port` | integer | `18789` | OpenClaw 官方默认 | 只允许 loopback Gateway 使用，不加入 UFW 放行端口 |
| `nuc_ops_agent_model` | string | `openai/gpt-5.6-terra` | 任务补充 | 同时写进 `model.primary` 与 `agents.defaults.models` 允许列表；取决于账号套餐 |
| `nuc_ops_agent_thinking_level` | string | `medium` | 任务补充 | `agents.defaults.thinkingDefault`；合法值 `off\|minimal\|low\|medium\|high\|xhigh\|adaptive\|max` |
| `nuc_timezone` | string | `Australia/Melbourne` | 5、13.2 | timer 与主机使用的时区 |
| `nuc_debian_codename` | string | `trixie` | 4、9.2 | 第三方 APT 仓库发行代号 |
| `nuc_architecture` | string | `amd64` | 4、9.2 | APT 仓库架构 |

### 2.2 目录、基础防护与 SSH

| 变量 | 类型 | 默认值摘要 | 来源 | 说明 |
|---|---|---|---|---|
| `nuc_srv_directories` | list[dict] | 见 3.1 | 9.1、11.1 | 每项固定含 `path`、`owner`、`group`、`mode`、`purpose`、`managed_by` |
| `nuc_debian_mirror_uri` | string | `http://deb.debian.org/debian` | 6.1 | Debian 主仓库；`base` 以 deb822 写入 `/etc/apt/sources.list.d/debian.sources` |
| `nuc_debian_security_uri` | string | `http://security.debian.org/debian-security` | 6.1 | 安全仓库；缺了它 unattended-upgrades 等于没开 |
| `nuc_debian_components` | list[string] | `main`、`contrib`、`non-free-firmware` | 6.1 | 与安装介质一致；`non-free-firmware` 必须保留，无线网卡等固件包在其中 |
| `nuc_base_packages` | list[string] | `openssh-server`、`curl`、`ca-certificates`、`gnupg`、`git`、`tmux`、`ripgrep`、`jq`、`ufw`、`unattended-upgrades`、`nvme-cli`、`smartmontools`、`lm-sensors`、`btop`、`restic` | 6.1 | Debian 基础包集合 |
| `nuc_sleep_targets` | list[string] | 四个 sleep target | 6.3 | `sleep.target`、`suspend.target`、`hibernate.target`、`hybrid-sleep.target` |
| `nuc_unattended_upgrades_auto_reboot` | boolean | `false` | 6.4 | 第一阶段必须保持关闭 |
| `nuc_journald_system_max_use` | string | `1G` | 任务补充、12.4 | journald 磁盘上限；PDF 未给数值，采用保守默认值 |
| `nuc_nvme_device` | path | `/dev/nvme0` | 12.4 | smartctl/nvme-cli 监控目标 |
| `nuc_ufw_lan_ssh_port` | integer | `22` | 7.2 | LAN SSH 端口 |
| `nuc_ufw_tailscale_ports` | list[integer] | `[22, 6767]` | 7.2、7.3 | `tailscale0` 显式允许端口；第二项必须等于 `nuc_paseo_port` |
| —（无变量） | — | — | 7.2、7.3 | 另有一条 `deny in from nuc_lan_cidr to nuc_paseo_port/tcp`，复用上述两个变量，不新增契约变量 |
| `nuc_sshd_hardening_options` | dict | 见 PDF 7.1 | 7.1 | 禁 root、启公钥、禁密码/KbdInteractive/X11 |
| `nuc_confirm_key_removal` | boolean | `false` | 7.1、安全门禁 | 仅当 preflight 列出的现有公钥确认可删除时以 extra-var 临时设为 `true` |

### 2.3 软件版本与服务配置

PDF v1.4 只明确固定 Debian 13.6 与 Node.js 22 LTS。其余软件没有给出可信的精确构建号，因此默认值表示 PDF 指定的稳定发行通道；若部署方需要可复现构建，可把空字符串或 `latest` 改成仓库中真实存在的精确版本，role 必须尊重该值，不能在代码里另写版本。

| 变量 | 类型 | 默认值 | 来源 | 说明 |
|---|---|---|---|---|
| `nuc_docker_package_versions` | dict[string,string] | 五个 Docker 包均为 `""` | 9.2 | 空值表示 Docker stable 当前候选；非空时按 `包=版本` 安装 |
| `nuc_docker_log_max_size` | string | `10m` | 9.3 | 合并进现有 `daemon.json` |
| `nuc_docker_log_max_file` | string | `3` | 9.3 | Docker 要求字符串值 |
| `nuc_docker_default_host_binding` | string | `127.0.0.1` | 任务补充、9.3 | 发布容器端口的默认宿主绑定地址；同时写入 `ip` 与 `default-network-opts.bridge.com.docker.network.bridge.host_binding_ipv4` |
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
| `nuc_paseo_relay_enabled` | boolean | `false` | 任务补充 | 启用后 daemon 外连中继，绕过 UFW 与 Cloudflare Access；需按交互清单人工配对 |
| `nuc_cloudflared_package_version` | string | `""` | 10.3 | 空值表示 Cloudflare 仓库当前候选 |
| `nuc_cloudflared_enable_service` | boolean | `true` | 10.4 | Dashboard 已建 tunnel 且 vault token 已填后安装/启用 service |
| `nuc_cloudflared_config_dir` | path | `/etc/cloudflared` | 10.4 | token 文件所在目录，`0700 root:root` |
| `nuc_cloudflared_token_path` | path | `/etc/cloudflared/token` | 10.4 | tunnel token 文件，`0600 root:root`；unit 以 `--token-file` 读取 |
| `nuc_openclaw_version` | string | `2026.8.1-beta.3` | 任务补充 | npm 上与已复核 mode/SQLite approvals/policy schema 匹配的精确预发布版；升级前必须重审 |
| `nuc_ops_agent_enabled` | boolean | `false` | 任务补充 | 人工门禁；`site.yml` 以 role 级 `when:` 控制，为 `false` 时整个 role 不执行 |
| `nuc_ops_agent_observable_units` | list[string] | 见 defaults | 任务补充 | journal 与 unit 状态读取的 unit 白名单 |
| `nuc_ops_agent_observable_properties` | list[string] | 见 defaults | 任务补充 | `systemctl show` 允许读取的属性白名单 |
| `nuc_restic_status_file` | path | `/var/lib/agent-restic/last-run.status` | 任务补充 | 单行 `ok`/`failed` 加 ISO 时间戳 |
| `nuc_restic_status_helper` | path | `/usr/local/sbin/agent-restic-status` | 任务补充 | 状态写入的唯一入口，成功与失败两条路径共用 |
| `nuc_restic_package_version` | string | `""` | 6.1、12.2 | 空值表示 Debian 13.6 当前候选 |
| `nuc_copilot_package` | string | `@github/copilot` | 任务补充 | GitHub Copilot CLI 的 npm 包名 |
| `nuc_copilot_version` | string | `latest` | 任务补充 | 版本 pin；为 `latest` 时必须省略 `@version`，否则每次重装（陷阱第 11 条） |
| `nuc_restic_s3_region` | string | `auto` | 任务补充 | S3 类后端的区域；Cloudflare R2 要求固定为 `auto`，AWS S3/MinIO 填各自真实区域。仅在配了 S3 凭据时写入环境文件 |

### 2.4 agent-runner、ops_agent 与 Restic

| 变量 | 类型 | 默认值 | 来源 | 说明 |
|---|---|---|---|---|
| `nuc_agent_runner_task_prompt` | string | `执行每日自动化任务，并把结果写入 reports/` | 11.3 | 第一阶段占位文案；必须替换成具体任务后才允许启用 timer |
| `nuc_agent_runner_config_dir` | path | `/etc/agent-runner` | 任务补充 | `root:agent-runner 0750`；位于 `ReadWritePaths` 之外 |
| `nuc_agent_runner_prompt_path` | path | `/etc/agent-runner/task-prompt.txt` | 任务补充 | `root:agent-runner 0640`；经 `StandardInput=file:` 送入 `codex exec -` |
| `nuc_agent_runner_schedule` | string | `*-*-* 07:30:00 Australia/Melbourne` | 11.3 | systemd timer `OnCalendar` |
| `nuc_agent_runner_randomized_delay` | string | `10m` | 11.3 | timer 随机延迟 |
| `nuc_agent_runner_timer_enabled` | boolean | `false` | 11.3 | 第一阶段保持 `false`；只有具体 prompt 经人工 service 验收后才改为 `true` |
| `nuc_restic_repository` | string | `local.yml` | 12.2 | 仓库路径或不含嵌入凭据的远端 URL；会暴露备份位置，不写入 00-contract.yml |
| `nuc_restic_backup_paths` | list[path] | `/srv`、`/etc/ssh`、`/etc/systemd/system`、ops 的 `MEMORY.md`/`memory`/`reports` | 12.1、12.2、任务补充 | 每晚备份范围；不备份可由 Ansible 重建的 ops 指令与 policy |
| `nuc_restic_excludes` | list[string] | 见下 | 8.2、12.2 | 排除缓存、worktree、依赖与 Codex 登录态 |
| `nuc_restic_keep` | dict | `{daily: 7, weekly: 4, monthly: 6, yearly: 1}` | 12.2、12.3 | `forget --prune` 保留策略 |
| `nuc_restic_schedule` | string | `*-*-* 02:30:00 Australia/Melbourne` | 12.3 | nightly timer |
| `nuc_restic_randomized_delay` | string | `30m` | 12.3 | timer 随机延迟 |
| `nuc_restic_read_data_subset` | string | `5%` | 12.2、12.4 | 人工/月度完整性检查读取比例 |
| `nuc_restic_env_file` | path | `/etc/restic/agent-nuc.env` | 12.2、12.3 | root-only 环境文件 |
| `nuc_restic_wrapper_path` | path | `/usr/local/sbin/agent-restic-backup` | 任务补充、12.2 | 串行执行 backup 与 forget 的包装脚本 |
| `nuc_restic_initialize_repository` | boolean | `false` | 任务补充、12.2 | **默认禁止自动 init**：地址写错时 init 会在错误位置建出一个全新的有效仓库，此后备份全部「成功」而历史快照不在其中。首次建库需人工临时置 true |
| `nuc_restic_expected_repository_id` | string | `""` | 任务补充 | `restic cat config` 的仓库 ID；非空时每次运行都校验，不一致即 fail。建议首次建库后写入 `local.yml` |
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

`group_vars/all/vault.yml.example` 只提交占位符；真实的 `vault.yml` 必须经 `ansible-vault` 加密并由 `.gitignore` 排除。同样地，`inventory.yml` 与 `files/preseed.cfg` 只提交 `.example` 模板，真实文件由 `.gitignore` 排除。

**模板文件的扩展名必须是 `.yml.example`，不得改成 `.example.yml`**：`group_vars/all/` 下扩展名落在 `YAML_FILENAME_EXTENSIONS`（默认 `['.yml', '.yaml', '.json']`）内的文件**全部会被 Ansible 加载**，模板也不例外。`local.example.yml` 这种命名会被当成真正的变量文件读进来，且字母序排在 `local.yml` 之前——真实文件覆盖它，看起来没问题，但**真实文件里漏填的项会静默落到模板的占位值上，而不是报 undefined variable**。已实测踩中：`vault.yml` 少写 `vault_nuc_ops_agent_gateway_token`，得到的不是失败而是安静的 `CHANGE_ME`。改成 `.yml.example` 后扩展名不在列表内，模板不再参与变量解析，漏填就会响亮地失败。

**加载顺序**：`group_vars/all/` 内的文件按**字母序**加载，后加载者覆盖先加载者。契约文件因此命名为 `00-contract.yml`，排在 `local.yml` **之前**，本机值天然覆盖契约默认值。**这个数字前缀是有意的，不得改名**。

此前该文件叫 `main.yml`，字母序排在 `local.yml` **之后**，于是契约文件反过来覆盖本机文件——与所有人的直觉相反。当时靠「`main.yml` 对本机变量只写注释、不赋值」这条纪律维持，而纪律先后被破三次并造成实际故障：`nuc_ops_agent_enabled`（门禁打不开）、`nuc_paseo_password_configured`（改了 local.yml 仍卡在门禁）、`nuc_restic_expected_repository_id`（被覆盖成空值，导致仓库身份校验被静默跳过，校验与它要防的故障一起消失）。改名后这一整类问题在结构上消失，不再依赖任何人记得那条纪律。

三层优先级由低到高：role 的 `defaults/main.yml` < `00-contract.yml` < `local.yml`。已实测确认。

| 变量 | 用途 | 来源 |
|---|---|---|
| `vault_nuc_restic_password` | Restic 独立随机长密码 | 12.2 |
| `vault_nuc_cloudflared_tunnel_token` | Dashboard 创建 tunnel 后得到的 service token | 10.4 |
| `vault_nuc_ops_agent_gateway_token` | loopback ops Gateway 的独立随机 token；目标机上通过 file SecretRef 读取 | 任务补充 |
| `vault_nuc_restic_s3_access_key_id` | S3 类后端（R2/S3/MinIO）的 Access Key ID；仅在 `nuc_restic_repository` 以 `s3:` 开头时必需 | 任务补充 |
| `vault_nuc_restic_s3_secret_access_key` | 对应的 Secret Access Key；与上一项同进同出 | 任务补充 |

下列内容不进入 Ansible Vault：管理员 `~/.codex/auth.json`、agent-runner
`CODEX_HOME` 与 ops OpenClaw state 内的 OAuth 登录缓存、Paseo 内层密码、个人 SSH
私钥和真实公钥指纹。三处 OpenAI 接入均使用 ChatGPT/Codex OAuth；本项目不接受、
存储或注入 `OPENAI_API_KEY`。

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
| `/var/lib/agent-runner/codex` | agent-runner | `0700` | `CODEX_HOME` 与 OAuth 登录缓存；不备份 | `agent_runner` |
| `/home/solar` | `solar:solar` | `0700` | ops 独立 home | `ops_agent` |
| `/home/solar/.openclaw` | `solar:solar` | `0700` | OpenClaw 可变 state 与认证资料 | `ops_agent` |
| `/var/lib/openclaw-ops-agent/workspace` | `root:solar` | `0750` | root 控制的 ops 工作区根 | `ops_agent` |
| `/var/lib/openclaw-ops-agent/workspace/AGENTS.md` | `root:solar` | `0640` | 每会话加载的不可变运行规则 | `ops_agent` |
| `/var/lib/openclaw-ops-agent/workspace/policy.jsonc` | `root:solar` | `0640` | OpenClaw conformance policy | `ops_agent` |
| `/var/lib/openclaw-ops-agent/workspace/MEMORY.md` | `solar:solar` | `0600` | 跨会话长期趋势 | `ops_agent` |
| `/var/lib/openclaw-ops-agent/workspace/{memory,reports}` | `solar:solar` | `0700` | 每日记忆与巡检报告 | `ops_agent` |
| `/etc/openclaw` | `root:solar` | `0750` | ops config、token 与 approvals 导入文件目录 | `ops_agent` |
| `/etc/openclaw/ops-agent.json` | `solar:solar` | `0600` | Ansible 管理的唯一 OpenClaw 配置 | `ops_agent` |
| `/etc/openclaw/ops-agent-gateway-token` | `solar:solar` | `0600` | Gateway token file SecretRef 来源；OpenClaw 拒绝 group-readable token，systemd 运行态仍把 `/etc` 设为只读 | `ops_agent` |
| `/etc/sudoers.d/90-ops-agent-smart` | `root:root` | `0440` | 仅两条固定设备、固定参数 SMART 命令 | `ops_agent` |
| `/etc/restic` | `root:root` | `0700` | Restic 配置目录 | `restic` |
| `/etc/restic/agent-nuc.env` | `root:root` | `0600` | 仓库地址与密码环境文件 | `restic` |
| `/etc/hostname` | `root:root` | `0644` | 系统主机名 | `base` |
| `/etc/hosts` | `root:root` | `0644` | 只维护 `127.0.1.1` 一行，其余保留 | `base` |
| `/etc/cloudflared` | `root:root` | `0700` | cloudflared token 目录 | `cloudflared` |
| `/etc/cloudflared/token` | `root:root` | `0600` | tunnel token；`no_log` 写入，不进 unit 文本与 cmdline | `cloudflared` |
| `/etc/systemd/system/cloudflared.service` | `root:root` | `0644` | 由 Ansible 管理，非 `cloudflared service install` 生成 | `cloudflared` |
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
| `ops_agent | daemon-reload system units` | `ops_agent` | system-level Gateway unit 变化 |
| `ops_agent | restart gateway` | `ops_agent` | 已验证的 config、policy、token 或 unit 变化 |
| `restic | daemon-reload system units` | `restic` | backup service/timer 变化 |

## 5. Tag 与执行顺序

role tag 与目录名完全相同，`site.yml` 只按下列顺序表达依赖，不使用 `meta/dependencies`：

| 顺序 | role/tag | 主要前置 |
|---:|---|---|
| 1 | `network` | 接口由 NetworkManager 管理；只改 profile 不激活，切换靠人工重启 |
| 2 | `base` | 裸 Debian 已能用管理员 + sudo 连接 |
| 3 | `ssh_harden` | `nuc_admin_authorized_keys` 非空且另一个会话已验证密钥 |
| 4 | `srv_layout` | 管理员账号存在 |
| 5 | `docker` | 基础网络与 ca-certificates 可用 |
| 6 | `codex` | 管理员账号存在；安装后人工 `codex login` |
| 7 | `tailscale` | 安装后人工 `sudo tailscale up`，再重跑完成地址校验 |
| 8 | `paseo` | Codex 已登录、Tailscale 地址已验证；config 写入后人工设密码 |
| 9 | `cloudflared` | Dashboard 已建 tunnel/Access，token 已写入 vault |
| 10 | `agent_runner` | Node/npm 可用；人工完成独立 Codex OAuth 登录；具体任务出现后才首次验收 service |
| 11 | `ops_agent` | Node/npm 可用；Gateway token 已写入 Vault；之后人工完成 ChatGPT/Codex OAuth 登录 |
| 12 | `restic` | 仓库地址和 vault 密码已填写 |
| 13 | `copilot` | Node.js 22 已由 `paseo` 装好；安装后人工 `copilot login --device-code`。**跑在管理员账号下**，与 `agent_runner`/`ops_agent` 那两个剥掉提权能力的 agent 账号不是同一类边界 |

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
- `restic` 的仓库存在性判定必须用 `restic cat config` 的退出码（10 = 仓库不存在，12 = 密码错误），不得匹配英文错误文本——文本随版本与 locale 变化。因此 role 必须先断言 restic ≥ 0.17.1，否则判定会静默失效。
- `cloudflared` 不得使用 `cloudflared service install`：该命令需要 `creates:` 才幂等，而 `creates:` 会让 unit 存在后永不重跑，vault 中轮换 token 后本机不再收敛。token 文件与 unit 一律由 Ansible 的 `copy`/`template` 管理，二者都 `notify` 重启；token 只走 `--token-file`，不得进入 `ExecStart` 文本或命令行。
- `docker` 必须读取并递归合并现有 `/etc/docker/daemon.json`，保留契约外已有键；不得整体覆盖。
- 升级 OpenClaw 的 npm 包**必须 notify Gateway 重启**。运行中的 Gateway 仍加载着旧安装，而 npm 已把那些文件换掉，随后任何请求都会报 `The running Gateway can no longer load part of its OpenClaw installation`。实测于 beta.2 → beta.3：role 报 `failed=0`、看起来一切正常，但 Gateway 直到手工重启前一直是坏的 —— handler 此前只挂在配置变更上，版本升级这条路径没有覆盖。
- `copilot` 刻意跑在**管理员账号**下，因为它是人工交互工具。管理员有 sudo，因此它与 `agent-runner`、`solar` 那两个刻意剥掉提权能力的账号**不是同一类边界**。若将来要让 Copilot 无人值守运行，必须先搬到专用账号并按 `agent_runner` 的模式约束，不得直接给 timer。其认证只走交互式 OAuth device flow，不得使用 `COPILOT_GITHUB_TOKEN`/`GH_TOKEN`/`GITHUB_TOKEN` 这类长期静态 token——与其余三处 agent 接入一致。本机没有运行中的 secret-service，token 会以明文落在 `~/.copilot/`；该目录在 `0700` 的管理员家目录内，其他 agent 账号无法进入，且 `/home` 不在 Restic 备份范围内。
- Paseo web UI 的连接表单默认值（`localhost` / `6767` / 不加密）在「经 Cloudflare 域名远程访问」这个场景下**三项全错**：对外端口是 443 不是 6767（Cloudflare 不暴露 origin 端口），必须勾 SSL，且必须填人工设置的 daemon 密码。参数不对时报错是 `无法连接到 tcp://localhost:6767 / code 1006` —— 那是表单回落到默认值后连本机失败的产物，**不指向真实原因**。排查这类问题必须用浏览器控制台的裸 WebSocket 走通全链路并读服务端关闭码（`4401 Password required` = 链路全通只差密码），不能只看 daemon 侧计数器：计数器为零同时符合「被中间层挡掉」与「客户端压根没发」两种情况。详见交互清单 C3。
- **restic 对不存在的备份路径返回退出码 0 并静默跳过**（实测 0.18.0）。因此备份包装脚本必须在调用 `restic backup` 之前逐个校验 `nuc_restic_backup_paths` 中的路径，缺失即失败。否则任何一个路径消失（目录改名、`ops_agent` 被关掉、role 调整）都会让备份继续每晚报 `ok`，而那部分数据已不在保护范围内——与「连错仓库、备份照样成功」同类。缩减备份范围时应同时修改 `nuc_restic_backup_paths`，而不是让脚本容忍缺失。
- 渲染 shell 脚本的 `template` 任务必须带 `validate: /bin/sh -n %s`。语法错误的脚本写进去之后，只有等 timer 真正触发时才暴露，届时只能看到 `status=2/INVALIDARGUMENT`，而备份可能已经连着几晚没跑。同类校验：sshd 配置用 `sshd -t`，`daemon.json` 用 `python3 -m json.tool`。
- `/etc/openclaw` 保持 `root:solar 0750`，**不得给 `solar` 目录写权限**。`solar` 拥有 `ops-agent.json` 本身，但没有目录写权限，因此 `openclaw models auth login` 在回写配置时会报 `EACCES ... ops-agent.json.lock`。这个报错是良性的：OAuth 资料存在 `OPENCLAW_STATE_DIR` 下的 sqlite 里，登录实际成功，role 的认证断言照常通过（已实测）。放开目录写权限会让 `solar` 能删除并重建 root 控制的 `ops-agent-exec-approvals.json`，用一道真实的安全边界换一个良性报错。
- **只读探测类的 `command`/`shell` 任务必须写 `check_mode: false`。** check 模式下被跳过的 command/shell 注册的是 `{'rc': 0, 'stdout': '', 'skipped': True, 'msg': 'Command would have run if not in check mode'}` —— `rc` 为 `0`、`stdout` 为空，**伪装成「命令成功且没有输出」**。下游只判断 `rc == 0` 会误认为探测通过，再拿空 `stdout` 去做格式校验就必然误报。已实测踩中两次：`codex` 报「`command -v codex` 没有返回绝对路径」、`tailscale` 报「没有返回唯一的 IPv4」，两者都让全量 `--check --diff` 无法跑完——而那正是第 4 节要求的幂等验证手段。只读探测加上 `check_mode: false` 后，check 模式会真正校验实际状态，而不是跳过校验。
- 已知的 check 模式假阳性（**不是缺陷，不要为消掉它们加代码**）：`get_url` 带 `force: false` 且目标已存在时仍报 `changed`（无法验证远端），`apt` 的 `update_cache` 同理。判断幂等以**实跑模式**的 `changed=0` 为准。
- Restic 的 S3 类凭据只写进 `nuc_restic_env_file`（`0600 root:root`，写入任务 `no_log`），**不得嵌入 `nuc_restic_repository`**。仓库地址会出现在日志、`systemctl show` 与备份状态里，嵌入密钥等于把它散播到这些地方。仓库以 `s3:` 开头时 role 必须先断言凭据非空——缺凭据时 restic 报的是含糊的认证/连接错误，指向网络或地址而非真正原因。
- 用 `ansible.builtin.stat` 判定**可执行文件**时必须写 `follow: true`。npm 全局安装与 Codex 官方安装脚本装出来的都是符号链接（实测 `/usr/local/bin/<pkg>` → `../lib/node_modules/…`，`~/.local/bin/codex` → `packages/standalone/current/bin/codex`），`stat` 默认不跟随链接会得到 `isreg=False`，而 `isreg` 判定会把一个完全可用的可执行文件判成缺失。apt 装的二进制是常规文件，加 `follow: true` 也无害。
- `base` 必须先写入 Debian 官方 APT 源再做任何 apt 操作。用完整 DVD ISO 离线安装且跳过网络镜像时，安装器会在装完后注释掉 cdrom 源，系统里一个可用源都不剩。
- **`cache_valid_time` 只比对 `/var/lib/apt/lists` 的目录 mtime，不看目录里有没有内容。** 空目录只要 mtime 够新（DVD 装机后正是如此），`apt` 模块就判定缓存有效、跳过 update 并报 `ok`，而列表自始至终是空的；随后 `full-upgrade` 报 `ok`、装包报 `No package matching 'curl' is available` —— 这条错误指向包名，和真正的原因完全对不上。因此 `base` 必须在 update 之后**实测**包列表非空，为空时以 `cache_valid_time: 0` 强制刷新，复查仍为空才失败。已实测复现并验证自愈。
- 任何会被渲染进配置文件的环境相关变量，其消费方 role 必须在写文件之前断言它不是占位值。`nuc_access_hostname` 由 `paseo` 拦（它进 `daemon.hostnames`，占位值会让 Access 路径的 Host 校验失败，而现象像「连接被拒绝」）；`nuc_hostname` 由 `base` 拦；两个 token 与 Restic 仓库/密码分别由 `cloudflared`、`ops_agent`、`restic` 拦。`nuc_tailscale_ipv4` 不需另拦——`tailscale` 已用实测地址与它比对。
- 域名类校验用带 `^...$` 锚点、只允许字母数字/连字符/点的正则即可，不要再叠加一条字符集正则去查引号和控制字符：Jinja 字符串字面量会把 `\\` 解成单个反斜杠，`[...\\]` 这种写法会变成未闭合字符集，正则编译直接报错。
- `network` 只修改实测已激活的 NetworkManager profile，**不得新建**：同一接口出现两个 autoconnect profile 时，开机激活哪一个取决于时序。`nuc_lan_connection_name` 与实测名不符时必须失败。
- `network` 必须保持 `conn_reload: false`（`community.general.nmcli` 的默认值），只执行 `nmcli con modify`。`modify` 不影响已激活的连接，因此 Ansible 连接不会在 play 中途断掉；地址切换是交互清单里的一道人工重启门禁。加上 `state: up` 或 `conn_reload: true` 会当场断连。
- 静态地址掩码只从 `nuc_lan_cidr` 推导，不得单设变量：UFW 规则用的是同一个 CIDR，两处分开写迟早出现 /22 与 /24 不一致，而这种错误要等到访问网段高位地址时才暴露。
- `collections/` 只装 `ansible.posix` 与 `community.general`，**没有 `ansible.utils`，`ipaddr` 系列过滤器不可用**。网段归属判定用纯 Jinja 整数运算（点分四段转 32 位整数后比较区间），不得引入新 collection。
- 已发布容器端口的流量在到达 UFW 的 INPUT 规则前就被 DNAT 转发，UFW 挡不住 `docker run -p`。`daemon.json` 必须同时设置 `ip`（默认 bridge）与 `default-network-opts.bridge.com.docker.network.bridge.host_binding_ipv4`（Compose 建的用户定义 bridge），只设其一覆盖不全。二者都是**默认值**，compose 中显式写 host IP 仍可覆盖；强制边界需要 `DOCKER-USER` 链规则，留待部署第一个 stack 时决定。
- `ssh_harden` 必须先落地 `authorized_keys`，再写 `10-hardening.conf`；写配置的任务必须带 `validate: /usr/sbin/sshd -t -f %s`。
- `nuc_admin_authorized_keys` 的每一项首段必须是 `nuc_ssh_supported_key_types` 中的类型：公钥比对只取前两段，带 options 的条目（`from=`、`command=`、`no-pty` 等）会比对错字段，在 `exclusive: true` 下表现为反复重写或误删。本项目明确不支持带 options 的条目，需要限制来源时用 sshd_config 的 `Match` 块表达。
- UFW 必须含 LAN SSH 规则、`tailscale0` 上的 22/6767，以及一条显式拒绝 `nuc_lan_cidr` 访问 `nuc_paseo_port` 的规则；接口未出现不影响先写规则。
- Paseo `config.json` 是唯一配置来源；unit 的 `ExecStart` 除 `daemon start --foreground` 外不得传配置参数。`daemon.listen` 固定 `0.0.0.0:<port>`，不绑定 `tailscale0` 地址——绑定会让 daemon 依赖 tailscaled 存活，把 Cloudflare Access 那条路径一并拖下水；unit 中也不得再加等待 tailscale 接口的 `ExecStartPre`。
- Paseo user unit PATH 必须含 `nuc_codex_admin_bin_dir` 的真实探测值；不得照抄固定 PATH。`loginctl enable-linger` 必须先于任何 `systemctl --user`，后者必须显式传 `XDG_RUNTIME_DIR=/run/user/{{ nuc_admin_uid }}`。
- agent-runner 的 Codex 必须安装为 `/usr/local/bin/codex`；`ProtectHome=yes` 下
  `HOME`/`CODEX_HOME` 指向 `/var/lib/agent-runner`，`ReadWritePaths` 只含
  `/srv/automation` 与 `/var/lib/agent-runner`。认证只允许独立 `CODEX_HOME` 中人工生成的
  ChatGPT/Codex OAuth 登录态；unit 不得注入 key，并必须用 `UnsetEnvironment` 清除
  `OPENAI_API_KEY` 与 `OPENAI_ADMIN_KEY`。
- agent-runner 的 `codex` 命令行有三处不可整段挪动的约束：`--ask-for-approval` 是**全局**参数，必须位于 `codex` 与 `exec` 之间（放在 `exec` 后当前 CLI 以 exit 2 拒绝，`Type=oneshot` 下等于每次触发必失败）；`--sandbox` 与 `--ephemeral` 是 `exec` 自己的参数，必须留在 `exec` 之后。
- agent-runner 的 prompt 不得内插进 `ExecStart`：systemd 把 `%` 当 specifier 前缀，prompt 中的双引号会提前终止参数。必须渲染到 `nuc_agent_runner_prompt_path` 并以 `StandardInput=file:` 送入 `codex exec -`。该文件须在 `ReadWritePaths` 之外，否则 agent 可以改写自己的任务指令。
- 对 agent-runner 的 Codex 约束（如 `forced_login_method`）必须用 `-c` 写在 unit 的命令行上，不得只写进 `CODEX_HOME/config.toml`——`CODEX_HOME` 在 `ReadWritePaths` 之内，那个文件只是建议值；root 拥有且在 `ProtectSystem=strict` 下只读的 unit 才是强制点。
- `ops_agent` 默认不启用（`nuc_ops_agent_enabled: false`）。它是唯一会把系统 journal 内容作为 tool result 送进模型上下文的组件，必须显式开启。
- `ops_agent` 读取 journal 与 unit 状态时，unit 必须来自 `nuc_ops_agent_observable_units` 白名单：确定性白名单优先于输出侧的正则脱敏，后者穷举不完日志中 secret 的形态，只能当兜底。不得使用 `systemctl status`——它默认附带最近 10 行 journal，会绕过白名单；机器读取一律用 `systemctl show` 并限定属性。
- `restic` 的 nightly service 必须有 `OnFailure=`，把失败写入 `nuc_restic_status_file`。该文件是**可查询的失败状态，不是告警**：在定义检查者、检查频率与通知出口之前，文档中不得称其为告警。
- agent-runner 不得加入 `sudo`、`docker` 或其他特权组，不得获得 Docker socket。
- `ssh_harden` 在 `exclusive: true` 写入前必须读取现有 `authorized_keys`，按每行前两段（类型 + key 主体）比较；发现移除项时必须在任何文件变更前失败，只有 `-e nuc_confirm_key_removal=true` 才能继续。
- `paseo` role 开头必须断言 `nuc_codex_admin_bin`、`nuc_codex_admin_bin_dir`、`nuc_tailscale_detected_ipv4` 都由本次 play 产生；缺失时提示使用 `--tags codex,tailscale,paseo`。
- OpenClaw 运行账号 `solar` 的附加组必须精确为 `systemd-journal`，不得属于 `sudo`、`docker`、`disk`；SMART 只能走 `/etc/sudoers.d/90-ops-agent-smart` 中设备和参数均写死的两条命令。
- ops Gateway 必须保持 `bind=loopback`、token file SecretRef、无 channel/webhook/mDNS、`tools.elevated.enabled=false`、`allowRealIpFallback=false`，且不得启用 Docker/browser sandbox 或 Skill Workshop。OpenAI provider 只允许 ChatGPT/Codex OAuth profile，systemd unit 必须清除 OpenAI key 环境变量；模型必须显式走 OpenClaw 原生 runtime，不启用 Codex plugin。`tools.exec.mode=ask` 只表达请求策略；SQLite host approvals 必须同时是 per-agent allowlist、`askFallback=deny`、`autoAllowSkills=false`。
- ops 的 `AGENTS.md` 与 policy 必须 root-owned；systemd 用 `ProtectSystem=strict` 把 Ansible 配置设为运行态只读。只允许 state、`MEMORY.md`、`memory/`、`reports/` 写入。

## 7. 禁止自动化与失败契约

下列步骤只能由人执行，必须写入 `docs/INTERACTIVE-CHECKLIST.md` 并标明位于哪个 tag 之前/之后。禁止 `expect`、`pexpect`、模拟按键或其他变通包装。

| 人工步骤 | 流程位置 | role 的允许行为 |
|---|---|---|
| BIOS 刷写及 BIOS 设置 | Ansible 之前（PDF 3、13.1） | 不探测、不代劳；清单记录验收项 |
| 管理员 `codex login` / `codex login --device-auth` | `codex` 安装后、`paseo` 前（8.1、8.5） | 只运行 `codex login status`；只有输出 `Logged in using ChatGPT` 才通过 |
| `sudo tailscale up` | `tailscale` 安装后、`paseo` 前（8.4、8.5、10.2） | 只运行 `tailscale ip -4`；无 100.x 或与配置不符时 `fail` |
| `paseo daemon set-password` | `config.json` 写入后、启用 user unit 前（8.5、8.6） | 以 `nuc_paseo_password_configured` 为人工确认门禁；为 `false` 时 `fail` |
| Cloudflare Dashboard 域名、Create tunnel、Published application、Access application/policy/MFA | `cloudflared` 安装 service 前（10.4、10.5） | 不调用 Dashboard/API；只消费 vault token 安装本机 service |
| 以 agent-runner 运行 `codex login --device-auth` | `agent_runner` 建目录/账号后、启用 automation 前（11.3） | 只运行 `codex login status`；只有 ChatGPT OAuth 登录才通过 |
| `openclaw models auth login --provider openai --device-code` | `ops_agent` 完成配置、审计和 Gateway 后 | Ansible 只读列出 profile；必须至少一个且所有 OpenAI profile 均为 `oauth` |

此外，第一阶段必须保持 automation timer 禁用。以后只有在占位 prompt 已替换成具体非确定性任务后，才能由人手工启动一次 service、检查 journal，再把 `nuc_agent_runner_timer_enabled` 改为 `true`。Restic 密码必须另存离线副本；该备份动作也只写入清单。

## 8. Subagent 写入边界

- 一个 subagent 只拥有一个 `roles/<name>/` 目录，不得修改其他 role 或任何项目级文件。
- subagent 可创建自己 role 内的 task/default/meta/handler/template/file；不得写 `site.yml`、`README.md`、`docs/INTERACTIVE-CHECKLIST.md`、`CONVENTIONS.md`、inventory 或 group_vars。
- 发现契约不足时必须回报主 agent，不得自行扩展公共变量或 handler。
