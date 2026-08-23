# ASUS NUC 13 Pro Debian Agent 节点 Ansible 项目

本项目把一台已经安装 Debian 13.6 (`trixie`, `amd64`) 的 ASUS NUC 13 Pro
收敛到部署指南第一阶段状态。附件实际版本为 **v1.4**；任务说明里的 v1.3 章节号已按
语义映射到 v1.4。跨 role 接口、变量和权限的唯一约定见 `CONVENTIONS.md`。

Ansible 不会代替人执行 BIOS、浏览器授权、交互式登录、Paseo 密码设置、
`systemd-creds` 输入或 Cloudflare Dashboard 配置。完整顺序见
`docs/INTERACTIVE-CHECKLIST.md`。

## 1. 控制端准备

建议在控制端使用 Python 3.12 虚拟环境：

```bash
python3.12 -m venv .venv
.venv/bin/pip install "ansible-core>=2.16,<2.17" ansible-lint
.venv/bin/ansible-galaxy collection install -r requirements.yml
```

项目固定了 `ansible.posix` 和仍兼容 `ansible-core` 2.16 的
`community.general` 版本。`collections/` 与 `.venv/` 均不会提交。

## 2. 首次使用：从模板生成本机配置

仓库中只提交结构、非环境相关的默认值和 `.example` 模板。所有与具体机器、网络、
账号、秘密相关的值都在不提交的本机文件里。克隆之后先生成这四个文件：

```bash
cp group_vars/all/local.example.yml group_vars/all/local.yml
cp group_vars/all/vault.example.yml group_vars/all/vault.yml
cp inventory.example.yml inventory.yml
cp files/preseed.example.cfg files/preseed.cfg
# 填入各自的真实值，然后
.venv/bin/ansible-vault encrypt group_vars/all/vault.yml
```

这四个文件都已被 `.gitignore` 排除。各自要填什么：

| 文件 | 填什么 |
|---|---|
| `group_vars/all/local.yml` | `nuc_admin_user`、`nuc_admin_authorized_keys`、`nuc_lan_cidr`、`nuc_tailscale_ipv4`、`nuc_access_hostname`、`nuc_restic_repository` |
| `group_vars/all/vault.yml` | `vault_nuc_restic_password`、`vault_nuc_cloudflared_tunnel_token` |
| `inventory.yml` | NUC 的 LAN 地址与 `ansible_user` |
| `files/preseed.cfg` | 管理员用户名、密码哈希、SSH 公钥 |

`group_vars/all/` 下的 yml 按字母序加载且后者覆盖前者，`local.yml` 排在 `main.yml`
之后，因此不需要额外配置。每一项的用途、实测命令与约束见 `main.yml` 顶部的注释，
变量清单以 `CONVENTIONS.md` 第 2 节为准。

`nuc_admin_authorized_keys` 必须至少包含一把**已经验证过能登录**的公钥，并且要覆盖
所有需要保留的公钥：`ssh_harden` 以 `exclusive: true` 写入 `~/.ssh/authorized_keys`，
未列出的公钥会在关闭密码认证的同一次运行中被移除。

### `files/preseed.cfg` 的两处凭据占位

`preseed.example.cfg` 里 `<<< >>>` 标出的两处必须自己生成，不要提交：

```bash
# 1. <<<PASSWORD_HASH>>> —— 管理员账号的密码哈希
#    mkpasswd 由 whois 包提供：sudo apt install whois
mkpasswd -m sha-512

# 2. <<<SSH_PUBLIC_KEY>>> —— 管理员账号的 SSH 公钥，整行照抄
ssh-keygen -t ed25519 -C "agent-nuc admin"   # 还没有密钥时先生成
cat ~/.ssh/id_ed25519.pub
```

`mkpasswd` 只把哈希写进 `preseed.cfg`，原始密码不进仓库。这里用的公钥必须与
`local.yml` 的 `nuc_admin_authorized_keys` 是同一把，否则 preseed 注入的公钥会在
`ssh_harden` 阶段被移除。

Cloudflare tunnel token 与 Restic 密码只放在加密的 `vault.yml` 中；管理员 Codex
登录态、Paseo 密码、SSH 私钥和 agent-runner API key 不得放入 Vault 或仓库。

先验证控制端文件，不连接目标机：

```bash
.venv/bin/ansible-lint site.yml roles
.venv/bin/ansible-playbook site.yml --syntax-check
```

再确认能够连接且 sudo 正常：

```bash
.venv/bin/ansible agent-nuc -m ansible.builtin.ping
```

## 3. 分阶段 Bootstrap

每一步都应对照交互清单。以下命令默认需要 Vault 密码：

```bash
.venv/bin/ansible-playbook site.yml --ask-vault-pass --tags base
.venv/bin/ansible-playbook site.yml --ask-vault-pass --tags ssh_harden
.venv/bin/ansible-playbook site.yml --ask-vault-pass --tags srv_layout,docker
```

在运行 `base` 前先确认 `nuc_lan_cidr`，否则启用 UFW 可能阻断 SSH。运行
`ssh_harden` 前，必须从第二个终端验证声明的公钥确实能登录；role 会先落地公钥，
再用 `sshd -t` 校验并写入加固配置。

随后依次跨过人工门禁：

1. 运行 `--tags codex`。首次安装后会在未登录时有意失败；以管理员身份执行
   `codex login`，再重跑同一 tag。
2. 运行 `--tags tailscale`。首次安装后会在未授权时有意失败；人工执行
   `sudo tailscale up`，把 `tailscale ip -4` 的唯一地址写入
   `nuc_tailscale_ipv4`，再重跑同一 tag。
3. 运行 `--tags codex,tailscale,paseo`。Paseo 写入 `config.json` 后会在
   `nuc_paseo_password_configured: false` 时有意失败；人工执行
   `paseo daemon set-password`，把该变量改为 `true`，再重跑这三个 tag。
4. 在 Cloudflare Dashboard 完成 tunnel、Published application 与 Access policy，
   将 token 写入加密 Vault，然后运行 `--tags cloudflared`。
5. 运行 `--tags agent_runner`。第一次会在创建账号、目录和 unit 后因缺少加密凭据
   而有意失败；按清单执行 `systemd-creds` 后重跑。手工验收 service 成功，再把
   `nuc_agent_runner_timer_enabled` 改为 `true` 并重跑。
6. 确认 Restic 仓库和 Vault 密码无误，运行 `--tags restic`，然后按清单立即执行
   一次 service 与完整性检查。

`nuc_codex_admin_bin_dir` 和 `nuc_tailscale_detected_ipv4` 是 play 内运行时事实，不会
持久化。因此配置或重跑 Paseo 时必须使用：

```bash
.venv/bin/ansible-playbook site.yml --ask-vault-pass \
  --tags codex,tailscale,paseo
```

不能只运行 `--tags paseo`。

## 4. Check mode 与幂等验证

每跨过一个交互门禁后，对相同阶段执行：

```bash
.venv/bin/ansible-playbook site.yml --ask-vault-pass \
  --tags <逗号分隔的阶段> --check --diff
```

随后再次实际执行相同 tag；第二次应无配置变更。全新主机上的完整 check mode 不会
真的安装软件或创建用户，因此依赖这些对象的后续检查可能被跳过；实际执行中的
`fail` 也可能是设计好的人工门禁，而不是幂等性故障。秘密相关 task 使用 `no_log`，
仍应避免在命令行、日志或提交中暴露真实值。

## 5. Role 顺序与安全边界

`site.yml` 固定按以下顺序执行，不使用 `meta/dependencies`：

```text
base → ssh_harden → srv_layout → docker → codex → tailscale → paseo
     → cloudflared → agent_runner → restic
```

- Paseo 监听 `0.0.0.0:6767`，relay 关闭；LAN 侧由 UFW 显式拒绝，tailnet 侧由
  `tailscale0` 的 allow 规则放行，公网入口由 Cloudflare Access 保护。cloudflared 与
  daemon 同机，origin 走 `http://127.0.0.1:6767`，不经过 tailnet 地址——因此
  tailscaled 挂掉只影响 tailnet 那一条路径，Access 路径不受牵连。
- `nuc_tailscale_ipv4` 仍然必需：它不在 `daemon.listen` 里，但必须在
  `daemon.hostnames` 里，否则 tailnet 客户端会被 Host 校验拒绝，且现象看起来像
  「连接被拒绝」。
- Docker 不把管理员或 agent-runner 加入 `docker` 组。
- `/srv/automation` 与 `/srv/workspaces` 平级，且只由 agent-runner 拥有。
- agent-runner 的 Codex 位于 `/usr/local/bin`，凭据只由 systemd 加密凭据加载。
- unattended-upgrades 已启用，但自动重启保持关闭。

最终验收项目和故障定位命令见 `docs/INTERACTIVE-CHECKLIST.md`。

## 6. 灾难恢复顺序

NUC 整机丢失或重装时按以下顺序恢复。前三步都不依赖 Restic，这是刻意的：Restic
仓库密码本身不在 NUC 上，也不在本仓库里，否则会形成「要恢复备份得先有备份」的
循环依赖。

```text
1. 用 files/preseed.cfg 重装 Debian 13.6
2. 在控制端 clone 本 repo
3. 从离线副本还原 group_vars/all/local.yml、vault.yml、inventory.yml
4. 按第 3 节分阶段跑 playbook，跨过各个人工门禁
5. 最后用 Restic 恢复 /srv/data
```

第 1–4 步只需要离线保管的凭据（密码哈希、SSH 私钥、Restic 仓库地址与密码、
Cloudflare token），把机器重新收敛到第一阶段状态；数据恢复是最后一步，此时
`restic` role 已经配置好仓库和环境文件，可以直接：

```bash
sudo systemd-run --wait --pipe --collect \
  --property=EnvironmentFile=/etc/restic/agent-nuc.env \
  /usr/bin/restic restore latest --target / --include /srv/data
```

必须离线保管、且不只有一份的内容见 `docs/INTERACTIVE-CHECKLIST.md`：Restic 仓库
地址与密码、管理员 SSH 私钥、Vault 密码。丢失 Restic 密码即所有备份无法解密。
