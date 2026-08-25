# ASUS NUC 13 Pro 第一阶段交互检查清单

本清单对应部署指南 **v1.4**。凡标为“人工”的步骤都不得由 Ansible、`expect`、
`pexpect`、模拟按键或其他包装代劳。命令中的尖括号占位符必须替换为实际值：
`<admin>`、地址、域名和 token 按实际环境填写；写成变量名的占位（如
`<nuc_restic_read_data_subset>`）取该变量在 `group_vars` 中的当前值，不要照抄
契约默认值。

## A. Ansible 之前：硬件、Debian 与初始连接

- [ ] **人工：更新 BIOS**（PDF 3、13.1）。使用 ASUS EZ Flash 3 和 F7 进入 BIOS；
  部署指南记录的基线为版本 0044，若 ASUS 已发布兼容的新版本，应按发布说明确认后
  使用当前兼容版本。
- [ ] **人工：设置 BIOS**（PDF 3）：UEFI 启动、Secure Boot，以及断电恢复后的
  `Restore AC Power Loss`/等价选项设为自动开机。保存设置并进行一次实际断电恢复测试。
- [ ] **人工：安装 Debian 13.6**（PDF 4、5）。确认主机名、时区、磁盘布局、管理员
  账号和 sudo 能力符合指南；该项目不自动安装操作系统。
- [ ] 从 NUC 执行 `ip route`，把实际家庭 LAN 网段写入 `nuc_lan_cidr`；把 LAN 地址与
  管理员账号写入 `inventory.yml`。在运行 `base` 前完成此项，避免 UFW 锁死 SSH。
- [ ] 从控制端验证 `ssh <admin>@<LAN-IP>` 与 `sudo -v` 均成功。
- [ ] 按 `README.md` 第 2 节从 `.example` 模板生成 `group_vars/all/local.yml`、
  `group_vars/all/vault.yml`（加密）、`inventory.yml`、`files/preseed.cfg` 并填完，
  再在控制端通过 lint 与 syntax-check。

## B. 基础防护、SSH 与本地服务

### B1. `base` 前后

- [ ] 运行 `--tags base`。
- [ ] 检查 `sudo ufw status verbose`：默认拒绝入站；LAN CIDR 只允许 TCP/22；
  `tailscale0` 允许 TCP/22 和 TCP/6767；另有一条显式 `DENY IN` 挡住
  LAN CIDR 访问 TCP/6767。
  这条 deny 现在是 Paseo 端口在 LAN 侧的**唯一**隔离手段（daemon 监听
  `0.0.0.0`），必须逐条核对存在，不能只看默认策略。
- [ ] 检查四个 sleep target 已 masked、`unattended-upgrades` 已运行且自动重启为 false、
  `smartd` 和 journald 上限生效。

### B2. `ssh_harden` 之前（关键防锁死门禁）

- [ ] 把至少一把真实公钥写入 `nuc_admin_authorized_keys`；不得放入私钥或占位符。
- [ ] 确保该公钥已经存在于管理员 `~/.ssh/authorized_keys`，从**第二个终端**新建连接并
  验证仅靠公钥能成功登录；保持原会话打开。
- [ ] 运行 `--tags ssh_harden`，再从第三个新会话验证公钥登录。确认密码登录、root 登录
  均已关闭后才结束旧会话。
- [ ] 运行 `--tags srv_layout,docker`，检查 `/srv` 权限矩阵和 Docker 日志轮转配置。

## C. Codex、Tailscale 与 Paseo

### C1. `codex` 安装后、`paseo` 之前

- [ ] 运行 `--tags codex`。首次执行应安装并探测管理员 Codex 路径；未登录时 role 会
  有意停止。
- [ ] **人工：以 `<admin>` 身份执行 `codex login`**（PDF 8.1）。完成 ChatGPT 浏览器
  登录；不要复制 `~/.codex/auth.json`，也不要将它纳入备份。
- [ ] 执行 `codex login status` 确认登录，再重跑 `--tags codex`。

### C2. `tailscale` 安装后、`paseo` 之前

- [ ] 运行 `--tags tailscale`。首次执行应安装并启动 `tailscaled`；未授权时 role 会
  有意停止。
- [ ] **人工：执行 `sudo tailscale up`**（PDF 8.4、10.2），在浏览器完成授权。
- [ ] 执行 `tailscale ip -4`，确认只返回一个 `100.64.0.0/10` 地址；把它写入
  `nuc_tailscale_ipv4`，再重跑 `--tags tailscale`。
- [ ] 从另一台 tailnet 设备验证 `ssh <admin>@<100.x>`。

### C3. Paseo 配置落地后、user service 启用之前

- [ ] 运行 `--tags codex,tailscale,paseo`。前两个 tag 会重新导出 Paseo 所需的
  Codex 路径和 Tailscale 地址；首次执行在写入 `~/.paseo/config.json` 后会因密码门禁
  有意停止。
- [ ] 检查 `config.json`：`daemon.listen` 是 `0.0.0.0:6767`，`hostnames` 同时含
  `<100.x>` 与 `nuc_access_hostname`，`relay.enabled` 为 `false`。
  `hostnames` 里的 `<100.x>` 不可省：它已经不出现在 `listen` 里，但 tailnet 客户端
  仍用它做 Host 校验。删掉之后 tailnet 访问会被拒绝，而现象看起来像「连接被拒绝」，
  很容易误判成网络或防火墙问题。
- [ ] **人工：以 `<admin>` 身份执行 `paseo daemon set-password`**（PDF 8.5、8.6）。
  如果当前 shell 找不到命令，先把 `~/.local/npm/bin` 加入 `PATH`。密码不得写入 Vault、
  group vars、shell 历史或仓库。
- [ ] 将 `nuc_paseo_password_configured` 改为 `true`，重跑
  `--tags codex,tailscale,paseo`。
- [ ] 以管理员身份检查：

  ```bash
  systemctl --user status paseo --no-pager
  journalctl --user -u paseo --no-pager -n 100
  sudo ss -lntp | grep ':6767'
  ```

  `6767` 应当监听 `0.0.0.0`。这是预期结果，不是配置错误——LAN 侧的隔离由 B1 里那条
  UFW deny 规则负责，不再由 bind 地址负责。

- [ ] **故障树须知**：从一台未加入 tailnet 的 LAN 设备探测
  `<NUC 的 LAN IP>:6767`，预期仍然是连不上（UFW 默认 DROP，表现为超时而非
  connection refused）。**现象与改动前一致，原因已经不同**：改动前是内核层没在这个
  地址上监听，改动后是监听着但被过滤器挡住。排查时不要把「探测不通」当成
  「daemon 没起来」的证据，先看 `ss -lntp` 和 `ufw status verbose`。

## D. Cloudflare 外部入口：`cloudflared` 之前

- [ ] **人工：在 Cloudflare Dashboard 接入域名**（PDF 10.3-10.5）。
- [ ] **人工：Create tunnel**，选择 cloudflared connector，并保存 tunnel service token。
- [ ] **人工：创建 Published application**。公开 hostname 必须等于
  `nuc_access_hostname`，origin service 必须是 `http://127.0.0.1:6767`。
  **不要填 Tailscale 地址**：cloudflared 与 Paseo daemon 在同一台机器上，走 loopback
  即可。填 `<100.x>` 会让这条每天在用的路径平白依赖 tailscaled 存活——tailscaled
  一挂，tailnet 和 Access 两条路径同时断，人在外面无法补救。
- [ ] **人工：创建 Access self-hosted application 与 Allow policy**。只允许预期身份，
  启用身份提供方 MFA；不得创建 Bypass policy。
- [ ] 把 service token 只写入加密 `group_vars/all/vault.yml` 的
  `vault_nuc_cloudflared_tunnel_token`，然后运行 `--tags cloudflared`。
- [ ] **轮换 token 时同样只改 Vault 再重跑 `--tags cloudflared`。** token 文件与 unit
  都由 Ansible 管理，会收敛并触发重启。核对 `sudo stat -c '%a %U:%G'
  /etc/cloudflared/token` 为 `600 root:root`，且 token 没有出现在
  `/etc/systemd/system/cloudflared.service` 里。
  轮换后旧 token 建不了新连接，但**已建立的 connector 会继续运行到重启**——
  所以不重启就看不出是否真的换成功了，必须确认 handler 触发了重启。
- [ ] 检查 `systemctl status cloudflared --no-pager` 和 tunnel health；从未登录浏览器访问
  hostname 应先进入 Cloudflare Access，从允许身份登录后才到 Paseo。
- [ ] 确认路由器没有给 NUC 配置公网端口转发。

## E. agent-runner：两个人工门禁

### E1. 账号、目录和 unit 落地后

- [ ] 运行 `--tags agent_runner`。第一次应创建受限账号、`/srv/automation`、
  `/var/lib/agent-runner/codex`、system-level Codex 和 systemd units，然后因凭据不存在
  而有意停止。
- [ ] **人工：执行以下命令**（PDF 11.3），粘贴 agent-runner 专用
  `OPENAI_API_KEY`，按 `Ctrl-D` 结束：

  ```bash
  sudo systemd-creds encrypt - /etc/credstore.encrypted/agent-codex-api-key
  ```

  不要把 API key 放入命令行、Vault、环境文件或仓库。检查凭据文件是
  `root:root` 且凭据目录为 `0700`。
- [ ] 重跑 `--tags agent_runner`。当 `nuc_agent_runner_timer_enabled: false` 时，role 会
  保持 timer 禁用并在人工验收门禁停止。

### E2. timer 启用之前

- [ ] **人工：首次运行并检查 service**：

  ```bash
  sudo systemctl start agent-codex-daily.service
  sudo systemctl status agent-codex-daily.service --no-pager
  sudo journalctl -u agent-codex-daily.service --no-pager -n 200
  ```

- [ ] 确认没有 `status=200/CHDIR`、`ProtectHome`、凭据加载或写权限错误，输出只写入
  `/srv/automation`。
- [ ] 执行 `id agent-runner`，确认没有任何附加组，尤其不属于 `sudo` 或 `docker`。
- [ ] 将 `nuc_agent_runner_timer_enabled` 改为 `true`，重跑 `--tags agent_runner`，再检查
  `systemctl list-timers agent-codex-daily.timer --all`。

## F. Restic：nightly timer 与恢复能力

- [ ] 为 `vault_nuc_restic_password` 生成独立随机长密码，并**人工保存一份离线副本**。
  `nuc_restic_repository` 不得在 URL 中嵌入账号、密码或 token。
- [ ] 核对备份范围、排除项和保留策略，再运行 `--tags restic`。当
  **`nuc_restic_initialize_repository` 默认为 `false`，role 不会自动建库。**
  这是刻意的：地址写错时 `restic init` 会在错误位置建出一个全新的、完全有效的
  仓库，之后每次备份都报成功，而你的历史快照一个都不在里面——直到需要恢复
  那天才会发现。
- [ ] **首次建库**：确认地址无误后，把 `nuc_restic_initialize_repository` 临时设为
  `true` 跑一次，然后**立刻改回 `false`**。
- [ ] **记录仓库 ID**。首次建库后 role 会在输出里打印当前仓库 ID；把它写入
  `group_vars/all/local.yml` 的 `nuc_restic_expected_repository_id`。此后每次运行
  都会校验连上的确实是这个仓库，地址被改动或指向了邻近路径上的另一个仓库时会
  立即失败而不是静默换库。ID 也应抄进离线凭据清单。
- [ ] 若 role 报「退出码 12 / 密码错误」，**不要**打开自动 init —— 仓库是存在的，
  用错误的密码 init 会在同一位置留下第二个空仓库。先核对 Vault 与离线副本。
- [ ] **人工：立即运行一次 nightly service 并检查日志**：

  ```bash
  sudo systemctl start agent-restic-backup.service
  sudo systemctl status agent-restic-backup.service --no-pager
  sudo journalctl -u agent-restic-backup.service --no-pager -n 200
  sudo systemctl list-timers agent-restic-backup.timer --all
  ```

- [ ] **人工：执行完整性检查**，并至少每月重复一次。读取比例取
  `nuc_restic_read_data_subset` 的当前值（契约默认 `5%`）；改过该变量就按改后的值执行，
  否则这一步验收的范围和配置对不上：

  ```bash
  sudo systemd-run --wait --pipe --collect \
    --property=EnvironmentFile=/etc/restic/agent-nuc.env \
    /usr/bin/restic check --read-data-subset=<nuc_restic_read_data_subset>
  ```

- [ ] 从仓库恢复一个测试文件到临时目录并核对内容；没有演练过恢复的备份不算验收完成。

## G. 第一阶段最终验收

- [ ] 重启后，所有 masked sleep target、UFW、`tailscaled`、Paseo、cloudflared 和两个
  timer 都保持预期状态。
- [ ] 实际断电再上电后，BIOS 自动开机；Debian、网络和受控服务恢复。
- [ ] LAN 与 Tailscale SSH 均能使用公钥；root/密码登录保持关闭。
- [ ] `ss -lntp` 显示 Paseo 只监听 Tailscale IPv4；relay 关闭。
- [ ] 路由器和主机没有公网入站端口，外部访问必须经过 Cloudflare Access；MFA 与
  Allow policy 生效且没有 Bypass。
- [ ] Docker `daemon.json` 保留已有键，并启用 `json-file` 的 `10m`/`3` 日志轮转。
- [ ] `/srv/automation` 与 `/srv/workspaces` 平级；agent-runner 无 sudo、Docker socket
  或管理员 home 访问权。
- [ ] Restic 最近一次备份、`check --read-data-subset=<nuc_restic_read_data_subset>`
  和测试恢复均成功，离线密码可用。
- [ ] 对已跨过所有人工门禁的完整配置运行 `--check --diff`，无意外变更。

## H. 离线凭据清单

以下内容必须存在 NUC 之外，且不只有一份。它们同时是 `README.md` 第 6 节灾难恢复顺序的前置条件：
没有这些，重装后的机器无法重新收敛，备份也无法解密。

- [ ] Restic 仓库 ID（`nuc_restic_expected_repository_id`）
- [ ] Restic 仓库地址与密码（`nuc_restic_repository`、`vault_nuc_restic_password`）
- [ ] Ansible Vault 密码（解密 `group_vars/all/vault.yml` 用）
- [ ] `group_vars/all/local.yml` 与 `inventory.yml` 的内容，或它们的备份位置
- [ ] 管理员 SSH 私钥，以及 `files/preseed.cfg` 里那个账号的密码
- [ ] 随上述内容一并记录：日期、机器标识（`agent-nuc`）、以及这些凭据用来做什么的
      一句话说明

## I. 设备失窃处置清单（PDF 5.3）

NUC 物理丢失或疑似被入侵时立即执行，顺序无强制依赖但都要做完：

- [ ] 从 tailnet 删除该 NUC 设备
- [ ] 撤销 Cloudflare Access 会话与设备授权
- [ ] 轮换该节点的 OpenAI/Codex key（管理员登录态与 agent-runner 的
      `/etc/credstore.encrypted/agent-codex-api-key` 两处）
- [ ] 更换 Restic repo 密码并同步离线副本
- [ ] 检查并轮换所有 Git 远端 deploy key
- [ ] 轮换 Cloudflare tunnel token 与 Ansible Vault 密码
