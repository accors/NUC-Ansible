# OpenClaw ops agent 安全配置复核

研究日期：2026-08-23。依据 OpenClaw 仓库提交 `3cb52f4bb869959dcd06cb6d4d33e34db3b6a665` 的官方文档；链接均固定到该提交。npm `latest` 当日仍为 `2026.7.1-2`，不支持这里使用的新 schema；生成配置另以当日已发布且 schema 匹配的 `2026.8.1-beta.2` CLI 实测通过，因此 role 精确 pin 该 beta，而不使用不存在的 `2026.8.1` stable。

## 结论摘要

- OpenClaw 主配置的规范入口是 `tools.exec.mode: "ask"`（解析为 `allowlist/on-miss`）；不能再与旧的 `tools.exec.security` / `tools.exec.ask` 混写。host approvals 则仍使用 `security: "allowlist"`、`ask: "on-miss"`、`askFallback: "deny"`、`autoAllowSkills: false`。默认不能依赖，因为 Gateway/node host 的默认姿态是 `full/off`。
- exec 有两层策略：OpenClaw 请求配置（`tools.exec.*`）和 host approvals 文档；两层都必须收紧。allowlist 条目可带 `pattern` 与可选 `argPattern`，并可按 agent 隔离。
- 当前版本的本机 approvals 权威记录位于 `OPENCLAW_STATE_DIR/state/openclaw.sqlite` 的 singleton row；`openclaw approvals set --file` 是把 JSON5 导入 SQLite，不是让运行时持续读取该文件。
- `tools.profile: "messaging"` 不包含 ops 所需的 `exec`、`process`、`read`、`write` 与 memory tools，却会带入消息、会话、子智能体和隐式 `bundle-mcp`。若保留该 profile，必须用 `alsoAllow` 补齐诊断能力，并用 `deny` 明确切断上述出口。
- `openclaw security audit --fix` 不是“安全模式总开关”：它只做确定性的权限收紧、部分开放 group policy 转 allowlist，以及日志敏感信息重定向等有限修复；不会轮换 token、禁用 exec/tools、改变 Gateway bind/auth/network 暴露或删除插件/skills。
- `openclaw policy` 是合规检查层，不是运行时拦截器；可要求 approvals 文件存在、限制 security 模式、关闭 `autoAllowSkills`，并核对预期 allowlist。
- ops 的跨运行趋势记忆使用 `MEMORY.md` 与 `memory/*.md`（`sources: ["memory"]`）；`rememberAcrossConversations: false` 只关闭受保护的私有会话 transcript 跨对话召回，不会关闭这些文件型记忆。
- OpenAI provider 的认证固定为 ChatGPT/Codex device-code OAuth；role 会通过脱敏的 `models auth list --json` 拒绝缺失 OAuth 或混入 `api_key` 的 OpenAI profile。这里不能只靠 policy：官方明确说明 policy 不读取 per-agent SQLite credential store。OAuth profile 与 OpenClaw state 均不进入备份；疑似泄露时在 ChatGPT/OpenAI 账号安全后台人工 revoke 对应登录会话。见 [auth-profile policy 的证据边界](https://github.com/openclaw/openclaw/blob/3cb52f4bb869959dcd06cb6d4d33e34db3b6a665/docs/cli/policy.md#L197-L202)。
- 新 schema 会把默认 `openai/gpt-5.6-sol` 路由到 Codex runtime；本实现显式为该模型设置 `agentRuntime.id: "openclaw"`，继续使用 OpenAI provider 的 ChatGPT/Codex OAuth transport，而不为 ops 增加 Codex plugin。认证方式与 agent runtime 相互独立。Doctor 对禁用 Codex plugin 的 runtime route 会明确报错，见 [route 检查与修复边界](https://github.com/openclaw/openclaw/blob/3cb52f4bb869959dcd06cb6d4d33e34db3b6a665/src/commands/doctor/shared/codex-route-warnings.ts#L67-L86)。
- Skill Workshop 的自主模式默认是 `auto`，会创建并可能应用持久技能变更；只读 ops 将其显式设为 `off`，把 `skill_workshop` 工具加入 deny，并保留 `approvalPolicy: "pending"`。见 [Workshop 配置语义](https://github.com/openclaw/openclaw/blob/3cb52f4bb869959dcd06cb6d4d33e34db3b6a665/docs/tools/skills-config.md#L360-L377)。
- 独立 loopback、token-auth、无 channel、`elevated.enabled=false` 的 ops 设计符合威胁模型的分层边界；但“只读”仍依赖 OS 用户权限、工具策略和 approvals，不能只靠提示词。

## exec approvals 的确切语法

官方 CLI 接受 JSON5；本机 approvals 记录可通过 `openclaw approvals set --stdin` 或 `--file` 替换，远端分别用 `--gateway` 或 `--node <id|name|ip>`。`allowlist add|remove` 的 `--agent` 默认是 `"*"`，即作用于所有 agent。

```bash
openclaw approvals set --stdin <<'EOF'
{
  version: 1,
  defaults: {
    security: "deny",
    ask: "off",
    askFallback: "deny",
    autoAllowSkills: false
  },
  agents: {
    ops: {
      security: "allowlist",
      ask: "on-miss",
      askFallback: "deny",
      autoAllowSkills: false,
      allowlist: [
        { pattern: "/usr/bin/systemctl", argPattern: "^--failed$" },
        { pattern: "/usr/bin/journalctl", argPattern: "^--no-pager( |$)" }
      ]
    }
  }
}
EOF
```

这是基于官方 schema 示例的最小化示意；实际命令必须逐条按 ops 的诊断需求审阅。默认策略保持 deny/off，只有 `ops` agent 得到 allowlist/on-miss。官方定义 `deny` 阻止 host exec，`allowlist` 只运行匹配条目，`on-miss` 仅在未匹配时询问，`askFallback: deny` 在没有可用 UI 或超时时阻止，`full` 则无需审批。见 [exec approvals schema 与模式](https://github.com/openclaw/openclaw/blob/3cb52f4bb869959dcd06cb6d4d33e34db3b6a665/docs/tools/exec-approvals.md#L110-L145) 和 [规范 `tools.exec.mode`](https://github.com/openclaw/openclaw/blob/3cb52f4bb869959dcd06cb6d4d33e34db3b6a665/docs/tools/exec-approvals.md#L151-L200)。

主配置对应部分应写成：

```json5
{
  tools: {
    exec: {
      host: "gateway",
      mode: "ask",
      strictInlineEval: true,
      safeBins: [],
    },
  },
}
```

`mode` 与 legacy `security/ask` 同时出现会被 schema 拒绝。见 [exec config 与 mode 表](https://github.com/openclaw/openclaw/blob/3cb52f4bb869959dcd06cb6d4d33e34db3b6a665/docs/tools/exec.md#L88-L139)。

CLI 允许的字段枚举是 `--security deny|allowlist|full`、`--ask off|on-miss|always`、`--ask-fallback deny|allowlist|full`；`cautious` preset 正好是 `host=gateway, security=allowlist, ask=on-miss, askFallback=deny`，但仍应显式关闭 `autoAllowSkills` 并检查 agent allowlist。见 [CLI 参数与 preset](https://github.com/openclaw/openclaw/blob/3cb52f4bb869959dcd06cb6d4d33e34db3b6a665/docs/cli/approvals.md#L20-L40) 和 [cautious preset](https://github.com/openclaw/openclaw/blob/3cb52f4bb869959dcd06cb6d4d33e34db3b6a665/docs/tools/exec-approvals.md#L299-L316)。

## policy 检查应如何落地

为 ops 增加 policy 规则可把配置漂移变成可审计 finding：要求 SQLite approvals 证据存在；默认只允许 `deny`；ops agent 只允许 `allowlist`；`allowAutoAllowSkills: false`；allowlist 用 `expected` 精确核对 pattern 和可选 argPattern。缺失/无效 approvals 记录是“不可观测证据”，不会被当作通过；省略的 security 会继承运行时默认 `full`。见 [policy 的继承与证据](https://github.com/openclaw/openclaw/blob/3cb52f4bb869959dcd06cb6d4d33e34db3b6a665/docs/cli/policy.md#L440-L458) 和 [ops 类 scoped 示例](https://github.com/openclaw/openclaw/blob/3cb52f4bb869959dcd06cb6d4d33e34db3b6a665/docs/cli/policy.md#L460-L490)。

Policy plugin 配置在 `plugins.entries.policy.config` 下，`path` 是相对 agent workspace 的路径；因此本实现把 `policy.jsonc` 放在 root 控制的 workspace 根，而不是 `/etc` 的绝对路径。`expectedHash` 不是原始文件的 SHA-256，而是 CLI 计算的 policy 规范化 hash；Ansible 不能用普通文件 checksum 代替。本实现依靠 root ownership + systemd 运行态只读，并每次执行 `policy check`，避免写入错误 hash。见 [Policy plugin 配置](https://github.com/openclaw/openclaw/blob/3cb52f4bb869959dcd06cb6d4d33e34db3b6a665/docs/cli/policy.md#L606-L637)。

`2026.8.1-beta.2` 的 `policy check` 还有一个实测限制：`tools.exec.mode: "ask"` 已是 config 的规范入口，但 policy 的 `tools.exec.allowSecurity/requireAsk` 证据仍读取被省略的 legacy 字段并按默认 `full/off` 报告，未使用 mode 的归一化结果。主 config 又明确禁止 mode 与 legacy 字段并存。因此本实现不写这两个会误报的 policy 规则；`allowHosts=gateway` 仍由 policy 检查，`mode=ask` 由 config schema 与 `approvals get --json` 的 effective policy 验证，host 层再由精确 approvals 强制。这是 beta CLI 的实测兼容处理。

同一 beta 的 `security audit --deep` 还有一个可复现的 live-probe 限制：`probeGateway` 以 `client.id="openclaw-probe"`、`mode="probe"` 连接，而 WebSocket 握手只为本机 `client.id="cli"`、`mode="cli"` 的共享 token/password 保留自报 scopes；无设备身份的其他客户端会清空 scopes。因此即使提供正确 token，fresh state 的深度审计仍会得到且只得到 `gateway.probe_failed: missing scope: operator.read`。这不是 Gateway 不健康或 token 错误：同一实例的 `openclaw health --json` 成功，Gateway 日志显示连接已建立后仅详情 RPC 因 scope 被拒。role 只允许这一条完全匹配的 beta warning，并另跑 health；任何其他 warning 都失败，未来版本修复后零 warning 也通过。相关实现见仓库提交中的 [`probeGateway` 客户端身份](https://github.com/openclaw/openclaw/blob/3cb52f4bb869959dcd06cb6d4d33e34db3b6a665/src/gateway/probe.ts#L439-L453) 与 [`shouldPreserveLocalCliSharedAuthScopes`](https://github.com/openclaw/openclaw/blob/3cb52f4bb869959dcd06cb6d4d33e34db3b6a665/src/gateway/server/ws-connection/handshake-auth-helpers.ts#L260-L276)。

`doctor --lint` 还会在 stderr 重复打印一条 profile advisory：配置了 `tools.fs.workspaceOnly`，却没有把 `edit` 加入 `alsoAllow`。这是有意边界而非 finding：ops 只显式获得 `read`/`write` 来维护 memory/reports，`edit` 同时在 `deny` 中；为消除提示而扩大 `alsoAllow` 会让配置意图更混乱。实测 JSON 仍为 `ok:true`、`findings:[]`，policy 又精确核对 `alsoAllow` 与 `denyTools`。role 接受这条已记录 advisory，但 `doctor` 的 error finding、普通 security audit 的任何 warning、以及 deep audit 除上述精确 probe warning 外的所有 warning 都会阻断部署。

不带 `--severity-min error` 跑完整 doctor 时，固定 beta 还会报告两条已复核 warning：一条建议为显式禁用的 `heartbeat: 0m` 落一条 disabled cron monitor 记录，另一条指出全局 host approvals 默认 `deny/off` 比 ops 的 per-agent `allowlist/on-miss` 更严格。Gateway 日志与 health 的 `everyMs: null` 证明 heartbeat 没有运行；`exec-policy show` 则证明 ops 的有效交集仍为 `allowlist/on-miss`、fallback deny。运行 `doctor --fix` 会触及超出这两项的配置修复，因此本 role 只以 error severity 作为 doctor 门禁，并保留更严格的全局 exec 默认值。禁用 cadence 仍投影为 disabled monitor 的实现见 [heartbeat monitor 解析](https://github.com/openclaw/openclaw/blob/3cb52f4bb869959dcd06cb6d4d33e34db3b6a665/src/cron/heartbeat-monitor.ts#L26-L64)。

重要边界：`policy check` / `doctor --lint` 只读；policy 不是每次 tool call 的执行拦截器。`doctor --fix` 只有在显式启用 `workspaceRepairs` 时才修改 policy-managed workspace settings。见 [policy repair 说明](https://github.com/openclaw/openclaw/blob/3cb52f4bb869959dcd06cb6d4d33e34db3b6a665/docs/cli/policy.md#L1004-L1015)。

## `security audit --fix` 的真实作用

官方列出的自动修复包括：把常见 `groupPolicy="open"` 改成 `allowlist`，收紧 state/config、credentials、auth profiles、SQLite 和 include 文件权限，并在 Windows 使用 ACL reset。它不轮换 token/password/API key，不禁用 `gateway`、`cron`、`exec` 等工具，不改变 Gateway bind/auth/network 暴露，也不删除或重写 plugins/skills。见 [audit --fix 变更边界](https://github.com/openclaw/openclaw/blob/3cb52f4bb869959dcd06cb6d4d33e34db3b6a665/docs/cli/security.md#L121-L137)。

因此装完可以运行 `openclaw security audit --fix`，但必须把它当作文件权限和部分 ingress policy 的安全护栏；它不能替代手工确认 loopback、token auth、无 channel、exec allowlist、无 elevated。audit checks 目录还明确：普通 audit 已覆盖大多数检查，只有 plugin/skill code scan 与 live Gateway probe 属于 `--deep`；权限和 Gateway 暴露 finding 的 auto-fix 标记分别可在表中核对。见 [audit checks 总表](https://github.com/openclaw/openclaw/blob/3cb52f4bb869959dcd06cb6d4d33e34db3b6a665/docs/gateway/security/audit-checks.md#L10-L34)。

## 对独立 ops agent 设计的安全含义

威胁模型把 Channel Access、Session Isolation、Tool Execution、External Content、Supply Chain 分成独立信任边界；Gateway 的 token/auth、按 agent 的 tool policy、exec approvals 和 sandbox/host 是不同控制点。见 [MITRE ATLAS trust boundaries](https://github.com/openclaw/openclaw/blob/3cb52f4bb869959dcd06cb6d4d33e34db3b6a665/docs/security/THREAT-MODEL-ATLAS.md#L30-L97)。

据此，ops 的建议配置应保持 Gateway `bind: "loopback"` + token auth、关闭 mDNS、无 channel/webhook、`elevated.enabled: false`，OS 层使用独立用户且不加入 `sudo`/`docker`/`disk`；systemd-journal 只读组不等于可执行权限。任何需要 `smartctl` 的能力应通过参数固定的 sudoers 条目单独授予。由于 ops 不在 Docker 组且不需要非主会话，本实现把 `agents.defaults.sandbox.mode` 明确设为 `off`，再依靠 systemd、OS 权限、workspaceOnly 与 host approvals 做强制边界；这是对“配置 skeleton 中 non-main”与“明确不启用 Docker sandbox”冲突的保守取舍。上述是对官方边界模型的配置推论，不是 OpenClaw 自动保证的隔离。

## 建议验证顺序

1. `openclaw approvals get --json`，核对 SQLite 中 defaults 与 `ops` 的 effective policy。
2. `openclaw exec-policy show --json`，核对 requested、host、effective 三者没有意外变宽。
3. `openclaw security audit`、`openclaw security audit --deep`、`openclaw security audit --fix`，记录修复前后 finding；`--fix` 后再次 audit。
4. `openclaw policy check`，启用上述 `execApprovals` 规则，确保 allowlist 与 policy expected 完全一致。
