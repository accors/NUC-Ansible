# ASUS NUC 13 Pro 第一阶段交互检查清单

本清单对应部署指南 **v1.4**。凡标为“人工”的步骤都不得由 Ansible、`expect`、
`pexpect`、模拟按键或其他包装代劳。命令中的尖括号占位符必须替换为实际值：
`<admin>`、地址、域名和 token 按实际环境填写；写成变量名的占位（如
`<nuc_restic_read_data_subset>`）取该变量在 `group_vars` 中的当前值，不要照抄
契约默认值。

## A. Ansible 之前：硬件、Debian 与初始连接

### A1. 硬件、系统与首次连接

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
- [ ] 在 GitHub 仓库 **Settings → Advanced Security** 确认 Secret Protection
  （secret scanning）与 Push protection 均已启用；这只是提交前护栏，不能替代
  Vault、`.gitignore` 与人工复核。

### A2. `network` 之后：静态地址切换（人工重启门禁）

`network` role 只改写 NetworkManager profile，**不激活**——`nmcli con modify` 不影响
已激活的连接，所以跑完 playbook 时地址没变、Ansible 连接也没断。切换靠这一步的重启。

- [ ] **先确认静态地址在路由器 DHCP 池之外。**
  从路由器管理界面查出 DHCP 池的起止范围。`nuc_static_ipv4` 若落在池内，早晚会与新接入的
  设备撞地址，而故障现象是间歇性的，很难往这个方向想。池内的话要么换地址，
  要么在路由器上为它加一条 MAC 保留。
- [ ] 运行 `.venv/bin/ansible-playbook site.yml --ask-vault-pass --tags network`。
- [ ] 在 NUC 上回读 profile，确认写进去的是期望值（此时运行态仍是旧的 DHCP 地址，
  这是正常的）：

  ```bash
  nmcli -g ipv4.method,ipv4.addresses,ipv4.gateway,ipv4.dns connection show "Wired connection 1"
  ```

  应输出 `manual`、`<nuc_static_ipv4>/<nuc_lan_cidr 的前缀>`、`<nuc_lan_gateway>`
  和 `nuc_lan_dns` 中的地址。**掩码必须与 `nuc_lan_cidr` 的前缀一致**：网段是 `/22`
  却写成 `/24` 的话，高位那 3/4 个网段会被当成跨网段，走网关绕行或直接不通。
  role 由 `nuc_lan_cidr` 推导掩码正是为了堵住这一类不一致。
- [ ] **人工：重启 NUC**（`sudo reboot`）。切换在此发生。
- [ ] 重启后确认运行态：

  ```bash
  ip -4 addr show "$IFACE"     # 应为 nuc_static_ipv4 加 nuc_lan_cidr 的前缀
  ip route                     # default via nuc_lan_gateway，dev 为该接口
  getent hosts deb.debian.org  # DNS 能解析
  ```

  以太网默认路由的 metric 必须**小于** WiFi 的。反了的话默认路由会走 WiFi：
  机器仍然在线，但用的是 WiFi 那一侧的地址，现象是「能 ping 通网段但
  `ansible_host` 连不上」。
- [ ] **如果以太网没起来**：本机保留了 WiFi 作为兜底，仍可通过 WiFi 侧的 DHCP 地址
  登录，不必接键盘显示器。登录后用 `nmcli con show "$CONN"` 查错，或
  `nmcli con modify "$CONN" ipv4.method auto` 临时退回 DHCP。
  （`$CONN` 即 `nuc_lan_connection_name`。）
- [ ] 把 `inventory.yml` 的 `ansible_host` 改成 `nuc_static_ipv4` 的值，
  再验证 `.venv/bin/ansible agent-nuc -m ansible.builtin.ping` 成功。

## B. 基础防护、SSH 与本地服务

### B1. `base` 前后

- [ ] 运行 `--tags base`。
- [ ] 检查主机名：`hostnamectl --static` 等于 `nuc_hostname`，且 `/etc/hosts` 里
  `127.0.1.1` 那行指向同一个名字。**两者不一致时 sudo 会反复报
  `unable to resolve host`**，而那时你可能已经在跑 ssh_harden 了。
  注意主机名会出现在 SSH 提示符、journald 日志和 Tailscale 设备名里；
  Tailscale 侧的设备名会被转成小写。
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
- [ ] 如果 role 列出将被 `exclusive: true` 移除的现有公钥，逐一按 key 主体确认来源，
  补齐所有仍需保留的设备公钥。只有确认列出的条目都应删除时，才加
  `-e nuc_confirm_key_removal=true` 重跑；不得把这个变量长期设为默认 true。
- [ ] 运行 `--tags ssh_harden`，再从第三个新会话验证公钥登录。确认密码登录、root 登录
  均已关闭后才结束旧会话。
- [ ] 运行 `--tags srv_layout,docker`，检查 `/srv` 权限矩阵和 Docker 日志轮转配置。
- [ ] 检查 `/etc/docker/daemon.json` 里 `ip` 与
  `default-network-opts.bridge.com.docker.network.bridge.host_binding_ipv4`
  都是 `127.0.0.1`，且原有的契约外键没有丢失。
  **这只是默认值**：将来 compose 里写 `0.0.0.0:8080:80` 依然会覆盖它，而且
  UFW 拦不住——Docker 的 DNAT 发生在 UFW 规则之前。部署第一个 stack 之前
  要先定规矩：compose 一律显式写 `127.0.0.1:`，需要远程访问时再单独审。

## C. Codex、Tailscale 与 Paseo

### C1. `codex` 安装后、`paseo` 之前

- [ ] 运行 `--tags codex`。首次执行应安装并探测管理员 Codex 路径；未登录时 role 会
  有意停止。
- [ ] **人工：以 `<admin>` 身份执行 `codex login`**（PDF 8.1；无浏览器终端可用
  `codex login --device-auth`）。完成 ChatGPT/Codex OAuth 登录；不要复制
  `~/.codex/auth.json`，也不要将它纳入备份。
- [ ] 执行 `codex login status`，确认显示 `Logged in using ChatGPT`，再重跑
  `--tags codex`；任何 API key 登录都不通过 role 门禁。

### C2. `tailscale` 安装后、`paseo` 之前

- [ ] 运行 `--tags tailscale`。首次执行应安装并启动 `tailscaled`；未授权时 role 会
  有意停止。
- [ ] **人工：执行 `sudo tailscale up`**（PDF 8.4、10.2），在浏览器完成授权。
- [ ] 执行 `tailscale ip -4`，确认只返回一个 `100.64.0.0/10` 地址；把它写入
  `nuc_tailscale_ipv4`，再重跑 `--tags tailscale`。
- [ ] 从另一台 tailnet 设备验证 `ssh <admin>@<100.x>`。

### C3. Paseo 配置落地后、user service 启用之前

- [ ] **先把 `nuc_access_hostname` 填进 `local.yml`，再运行 paseo。** 它会被渲染进
  `daemon.hostnames`，Access 路径靠它做 Host 校验。D 节只重跑 `--tags cloudflared`，
  不会回头修 `config.json`，所以留着占位值跑过去的话，等到从 Access 访问时才会暴露，
  且现象看起来像「连接被拒绝」。这个值就是你打算用的域名，不必等 tunnel 建好；
  要等 Dashboard 的是 D 节的 token。paseo role 的第一条 task 会拦占位值和非法域名。
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

## E. agent-runner：OAuth 门禁与保持 timer 关闭

> **先确认命令行本身能跑通，再手工首次运行 service。** 这里出过一次
> `--ask-for-approval` 位置错误导致 `Type=oneshot` 每次触发即 exit 2 的问题，而
> `ansible-lint` 与 `systemd-analyze verify` **都逮不到**：前者只验 YAML 与风格，
> 后者只验 unit 语法，都不验目标二进制接不接受这些参数。
>
> ```bash
> systemctl cat agent-codex-daily.service
> # 把 ExecStart 原样抄出来、末尾加 --help 跑一次，确认不是 exit 2
> ```
>
> 同时确认 `/etc/agent-runner/task-prompt.txt` 是 `root:agent-runner 0640`——
> agent 必须读得到、但改不了自己的任务指令。

### E1. 账号、目录和 unit 落地后

- [ ] 运行 `--tags agent_runner`。第一次应创建受限账号、`/srv/automation`、
  `/var/lib/agent-runner/codex`、system-level Codex 和 systemd units，然后因独立 OAuth
  登录态不存在而有意停止。
- [ ] **人工：执行以下命令**（PDF 11.3），按页面提示完成 ChatGPT/Codex device-code
  OAuth 登录：

  ```bash
  sudo -u agent-runner -H env \
    CODEX_HOME=/var/lib/agent-runner/codex \
    /usr/local/bin/codex login --device-auth

  sudo -u agent-runner -H env \
    CODEX_HOME=/var/lib/agent-runner/codex \
    /usr/local/bin/codex login status
  ```

  第二条命令必须显示 `Logged in using ChatGPT`。本项目没有 agent-runner 的 OpenAI
  API key；`/var/lib/agent-runner/codex` 必须为 `agent-runner:agent-runner 0700`，其中的
  OAuth 登录缓存不得复制到 Vault、仓库或备份。
- [ ] 重跑 `--tags agent_runner`。当 `nuc_agent_runner_timer_enabled: false` 时，role 应
  安全完成并保持 timer disabled，不再为了占位 prompt 故意失败。

### E2. 将来确有具体任务时，timer 启用之前

- [ ] 第一阶段不要执行本小节。默认 prompt 只是占位，必须保持
  `nuc_agent_runner_timer_enabled: false`；能由 Ansible/systemd 确定性完成的周期任务
  继续写成 Ansible/systemd，只有需要非确定性判断的明确任务才交给 agent。
- [ ] 把 `nuc_agent_runner_task_prompt` 改成有输入、输出与验收标准的具体任务；role 会
  拒绝为空或仍等于默认占位文案的 prompt。

- [ ] **人工：首次运行并检查 service**：

  ```bash
  sudo systemctl start agent-codex-daily.service
  sudo systemctl status agent-codex-daily.service --no-pager
  sudo journalctl -u agent-codex-daily.service --no-pager -n 200
  ```

- [ ] 确认没有 `status=200/CHDIR`、`ProtectHome`、OAuth 登录或写权限错误，输出只写入
  `/srv/automation`。
- [ ] 执行 `id agent-runner`，确认没有任何附加组，尤其不属于 `sudo` 或 `docker`。
- [ ] 将 `nuc_agent_runner_timer_enabled` 改为 `true`，重跑 `--tags agent_runner`，再检查
  `systemctl list-timers agent-codex-daily.timer --all`。

## F. ops agent（运行账号 `solar`）：只读观察者与人工 OAuth 登录

> **默认不启用。** `nuc_ops_agent_enabled` 为 `false` 时 `site.yml` 会跳过整个 role。
> 这是本项目唯一一条把系统日志送出本机的路径（journal 内容会作为 tool result 进入
> 模型上下文），确认要用再显式打开。

- [ ] 用 `openssl rand -hex 32` 生成独立随机 token，只写入加密 `vault.yml` 的
  `vault_nuc_ops_agent_gateway_token`；该 token 不需要离线保存，灾难恢复时重新生成。
- [ ] 把 `nuc_ops_agent_enabled` 改为 `true`（默认 `false`，否则 role 整体跳过）。
- [ ] 核对 exec 白名单里的 journal 相关规则：`journalctl` 必须带 `-u` 且 unit 属于
  `nuc_ops_agent_observable_units`；**不得存在任何 `systemctl status` 规则**——
  它默认附带最近 10 行 journal，会绕过 unit 白名单。机器读取一律用
  `systemctl show` 并限定属性。新增被监控服务时要显式加进白名单。
- [ ] 运行 `--tags ops_agent`。role 应完成 OpenClaw 安装、精确 approvals 导入、policy
  检查、`security audit --fix`、普通/深度审计与 loopback Gateway 健康检查。
- [ ] `doctor --lint --severity-min error` 可能在 stderr 提示 `tools.fs` 不会隐式加入
  `edit`；这是已记录的有意边界：
  ops 只显式允许 `read`/`write`，而 `edit` 明确位于 deny。确认其 JSON 仍为
  `ok:true`、`findings:[]`；任何真正的 error finding 都不能接受。
- [ ] 不带 `--severity-min error` 单独运行 doctor 时，当前 beta 还会建议为已禁用的
  `heartbeat: 0m` 创建 disabled cron 记录，并提示全局 exec 默认 `deny/off` 比 ops 的
  per-agent `allowlist/on-miss` 更严格。前者不代表 heartbeat 在运行，后者由
  `exec-policy show` 的 ops effective scope 复核；不要为消除这两条 warning 运行会改动
  其他配置的 `doctor --fix`，也不要放宽全局默认值。
- [ ] 执行 `id solar`，确认 home 是 `0700`，附加组只有 `systemd-journal`，且绝不
  属于 `sudo`、`docker`、`disk`。确认 `/etc/sudoers.d/90-ops-agent-smart` 只有两条
  设备与参数都写死的 SMART 命令。
- [ ] 确认 `/etc/openclaw/ops-agent-gateway-token` 为 `solar:solar 0600`；OpenClaw
  会拒绝 group-readable 的 SecretRef 文件，运行中的 service 则由 `ProtectSystem=strict`
  阻止改写 `/etc`。
- [ ] 检查 `ss -lntp | grep ':18789'`：只能看到 loopback，不能看到 LAN 或 Tailscale
  地址；`openclaw-ops-agent.service` 必须由 systemd system unit 管理。
- [ ] **人工：以 `solar` 身份完成 OpenAI 模型登录**：

  ```bash
  sudo -u solar -H env \
    OPENCLAW_CONFIG_PATH=/etc/openclaw/ops-agent.json \
    OPENCLAW_STATE_DIR=/home/solar/.openclaw \
    /usr/local/bin/openclaw models auth login \
      --provider openai --method device-code --agent ops
  ```

  Ansible 不模拟交互式登录，也不复制其他账号的认证文件。完成后以相同环境运行
  `openclaw models auth list --provider openai --agent ops --json`，确认所有 OpenAI profile
  的 `type` 都是 `oauth`，没有 `api_key`，再重跑 `--tags ops_agent`。
- [ ] 用同一组 `sudo -u ... env` 前缀分别运行 `openclaw approvals get --json`、
  `openclaw exec-policy show --json`、`openclaw policy check --agent ops --json`、
  `openclaw security audit --deep --json`；
  确认 defaults 为 deny、ops 为 allowlist/on-miss、`askFallback=deny`、
  `autoAllowSkills=false`，且没有 critical finding。固定的 `2026.8.1-beta.2` 在全新 state
  上可能只出现 `gateway.probe_failed: missing scope: operator.read`：这是该 beta 的无设备
  `probe` 客户端清空自报 scope 所致，role 只精确接受这一项，并另跑 `openclaw health`
  验证 Gateway；其他 warning 一律失败。升级后该 warning 消失是正常结果。
- [ ] 用以下命令做最小功能测试；`uptime`、失败服务和固定 SMART 形状应允许，`id`、
  `systemctl restart ...`、任意 shell/解释器与未列出的参数必须被拒绝：

  ```bash
  sudo -u solar -H env \
    OPENCLAW_CONFIG_PATH=/etc/openclaw/ops-agent.json \
    OPENCLAW_STATE_DIR=/home/solar/.openclaw \
    /usr/local/bin/openclaw agent --agent ops \
    --message "运行 uptime，检查失败服务，然后尝试运行 id；逐项报告是否获准"
  ```

- [ ] 确认 `AGENTS.md` 与 `policy.jsonc` 为 root-owned，ops 只能写 `MEMORY.md`、
  `memory/`、`reports/`；journal 内容必须被当作不受信任的数据而非指令。
- [ ] 第一阶段不配置 channel、webhook、browser、MCP、Docker sandbox 或定时巡检。
  若以后验证单向 `--deliver`，必须先让目标 channel 的入站 DM 完全 disabled，并实测
  “可出站、回消息不触发 turn”；在实测前不得把该推论当成安全保证。
- [ ] 目标机没有 Ansible repo、Vault 或提权能力，因此暂不允许 ops 直接跑
  `ansible-playbook --check --diff`。以后只能通过 root 控制、参数固定、仅产出报告的
  wrapper 增加该能力，不能把 repo 凭据或宽泛 sudo 交给 ops。

## G. Restic：nightly timer 与恢复能力

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
- [ ] 记住备份状态的查询方式：`cat /var/lib/agent-restic/last-run.status`，
  内容是 `ok <ISO时间>` 或 `failed <ISO时间>`。**这不是告警**——没有检查者、
  没有检查频率、没有通知出口，只是把「上一次跑挂了」变成可查询的事实。
  在定义出谁看、多久看、从哪通知之前，不要把它当成会主动找你的东西。
- [ ] 手工触发一次失败（例如临时把 `nuc_restic_repository` 指向不存在的路径并
  跑一次 service），确认状态文件变成 `failed`，再恢复并确认变回 `ok`。
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

## H. 第一阶段最终验收

- [ ] 重启后，所有 masked sleep target、UFW、`tailscaled`、Paseo、cloudflared、
  OpenClaw ops Gateway 和两个 timer 都保持预期状态。
- [ ] 实际断电再上电后，BIOS 自动开机；Debian、网络和受控服务恢复。
- [ ] LAN 与 Tailscale SSH 均能使用公钥；root/密码登录保持关闭。
- [ ] `ss -lntp` 显示 Paseo 监听 `0.0.0.0:6767`（这是预期结果，不是配置错误）；
  ops Gateway 的 18789 只监听 loopback；Paseo relay 关闭。
- [ ] `ufw status verbose` 里 LAN 访问 6767 被 deny、tailnet 经 `tailscale0` 被 allow。
  **那条 deny 是 6767 在 LAN 侧唯一的隔离手段**，不像别处还有第二层。
- [ ] 路由器和主机没有公网入站端口，外部访问必须经过 Cloudflare Access；MFA 与
  Allow policy 生效且没有 Bypass。
- [ ] Docker `daemon.json` 保留已有键，并启用 `json-file` 的 `10m`/`3` 日志轮转。
- [ ] `/srv/automation` 与 `/srv/workspaces` 平级；agent-runner 无 sudo、Docker socket
  或管理员 home 访问权。
- [ ] OpenClaw 运行账号 `solar` 只有 journal 读取与两条固定 SMART sudoers；无 channel、Docker socket、
  `disk` 组或系统修改能力，`MEMORY.md`/`memory`/`reports` 已进入 Restic 范围。
- [ ] Restic 最近一次备份、`check --read-data-subset=<nuc_restic_read_data_subset>`
  和测试恢复均成功，离线密码可用。
- [ ] 对已跨过所有人工门禁的完整配置运行 `--check --diff`，无意外变更。

## I. 离线凭据清单

以下内容必须存在 NUC 之外，且不只有一份。它们同时是 `README.md` 第 6 节灾难恢复顺序的前置条件：
没有这些，重装后的机器无法重新收敛，备份也无法解密。

- [ ] Restic 仓库 ID（`nuc_restic_expected_repository_id`）
- [ ] Restic 仓库地址与密码（`nuc_restic_repository`、`vault_nuc_restic_password`）
- [ ] Ansible Vault 密码（解密 `group_vars/all/vault.yml` 用）
- [ ] 管理员 SSH 私钥（公钥在恢复时用 `ssh-keygen -y -f <private-key>` 重新导出）

随这三项一并记录日期、机器标识（`agent-nuc`）和用途说明即可；`local.yml`、
`vault.yml`、`inventory.yml`、preseed 密码哈希、Cloudflare token 与 ops Gateway token
都可按 `README.md` 第 6 节重新生成，不是额外离线保管对象。

## J. 设备失窃处置清单（PDF 5.3）

NUC 物理丢失或疑似被入侵时立即执行，顺序无强制依赖但都要做完：

- [ ] 从 tailnet 删除该 NUC 设备
- [ ] 撤销 Cloudflare Access 会话与设备授权
- [ ] 在 ChatGPT/OpenAI 账号安全后台手动 revoke 与该节点有关的 Codex OAuth 登录会话
      （管理员 Codex、agent-runner Codex、ops OpenClaw）；本配置没有可轮换的 OpenAI API key
- [ ] 更换 Restic repo 密码并同步离线副本 —— 磁盘未加密，必须假定盘上的
      `/etc/restic/agent-nuc.env` 已被读走，即整个备份仓库已暴露
- [ ] 若是换盘/RMA/转卖而非失窃：先 `nvme format -s1 /dev/nvme0n1` 再交出硬盘
- [ ] 检查并轮换所有 Git 远端 deploy key
- [ ] 轮换 Cloudflare tunnel token 与 Ansible Vault 密码
- [ ] 重新生成 `vault_nuc_ops_agent_gateway_token`；在可信重建主机上分别重新完成三处
      ChatGPT/Codex OAuth 登录，不复用丢失主机上的缓存
