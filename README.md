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

## 2. 填写清单、变量与 Vault

1. 修改 `inventory.yml` 中的 LAN 地址、管理员账号和 Python 路径。
2. 修改 `group_vars/all/main.yml` 中所有 `CHANGE_ME`、示例网络值和版本 pin。
   `nuc_admin_authorized_keys` 必须至少包含一把已验证的 SSH 公钥。
3. 创建加密 Vault：

```bash
cp group_vars/all/vault.example.yml group_vars/all/vault.yml
.venv/bin/ansible-vault encrypt group_vars/all/vault.yml
.venv/bin/ansible-vault edit group_vars/all/vault.yml
```

真实的 `group_vars/all/vault.yml` 已被 `.gitignore` 排除。Cloudflare tunnel token
与 Restic 密码只放在该加密文件中；管理员 Codex 登录态、Paseo 密码、SSH 私钥和
agent-runner API key 不得放入 Vault 或仓库。

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

- Paseo 只监听 Tailscale IPv4，relay 关闭；公网入口由 Cloudflare Access 保护。
- Docker 不把管理员或 agent-runner 加入 `docker` 组。
- `/srv/automation` 与 `/srv/workspaces` 平级，且只由 agent-runner 拥有。
- agent-runner 的 Codex 位于 `/usr/local/bin`，凭据只由 systemd 加密凭据加载。
- unattended-upgrades 已启用，但自动重启保持关闭。

最终验收项目和故障定位命令见 `docs/INTERACTIVE-CHECKLIST.md`。
