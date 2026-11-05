# 三端共享记忆 · 实况（2026-11-05 复盘后）

dsh / zcode / hermes 三端共享记忆体系。文档为**当前实况**（2026-11-05 核对），
与 `三端记忆共享方案.md`（初始落地记录）配合阅读。

> **2026-11-05 实况修正**（实测核对）：
> - 服务已迁到雷神机，地址是 **`http://10.126.126.3:9177`**（EasyTier mesh，只绑 mesh IP），本机 `127.0.0.1:9177` 不可达；旧本地服务/PG 已退役（`~/.hindsight/win-service/stop.flag`）。
> - 精选漏斗 cron 实际是 **`30 6 * * *`（06:30）**，不是 8:30；看门狗 `12 8,20 * * *`。
> - **DSH 侧的自动流水 hooks（方案 1a）当前未挂载**：desktop/headless/tui 三个 `cordis.patch.yml` 里没有 `hooks-hindsight` insert，`dsh-mnt-d-dshprojects` 自 2026-09-26 起无新写入。本目录 `hooks/retain.py` 仍是可用的实现，只是没有挂上去。
> - zcode hooks 仍在（`~/.zcode/hooks/hindsight/`），但 `~/.hindsight/zcode.json` 指向 `http://localhost:9177` → 已不可达，实际不再产生记忆。
> - DSH 新增「热记忆 + 语义记忆」两层接入方式，见下方 **DSH 侧（2026-11-05）** 一节。


## 架构：私有流水 + 精选共享

```
hermes 各 profile  ──自动每轮──▶ hermes-default / hermes-hermesatdeepseek / hermes-hermesforwiki / hermes（私有流水）
hermes 精选漏斗 cron ──每天 06:30──▶ shared（从各私有 bank 选近 26h 新增，本地 LLM 同义去重，≤10 条/bank/天，失败推运维群；shared 护栏 650 条，自精简冗余批次）
zcode             ──自动每轮──▶ zcode（私有流水，retainBankId=zcode）
zcode save_shared.py ──手动──▶ shared（agent:zcode）
dsh hooks(未挂载)  ──自动每步──▶ dsh-<workspace>（私有流水，UserPromptSubmit/PostToolUse）
dsh skill+AGENTS.md ──每会话──▶ 热记忆自动注入；dsh smem save ──手动按需──▶ shared（agent:dsh）
所有端 recall（hermes/zcode 自动、dsh 手动 hmem search）◀── shared（精选层）
```

要点：**shared 只放值得跨端共享的结论级内容**；每端全量流水留在各自私有 bank。

## 现状核对（2026-11-05 实测）

| 项 | 值 |
|---|---|
| 记忆服务 | 雷神机 Hindsight（mesh 内）：`http://10.126.126.3:9177`（hindsight-api 0.6.2 + Postgres 18/pgvector，无鉴权，只绑 EasyTier IP）；本机 `127.0.0.1:9177` 不可达 |
| banks（2026-11-05） | `hermes-default`(7183) · `hermes-hermesforwiki`(7124) · `dsh-mnt-d-dshprojects`(4049，自 09-26 停) · `shared`(634) · `hermes-atCompany-default`(652) · `hermes-hermesatdeepseek`(338) · `hermes-hermesforlocalevaluation`(88) · `hermes-atCompany-hermesforcomanylocal`(59)；`news`/`zcode`/`hermes`/`hermes-hermesatlocal` 已在 11-05 清理 |
| hermes | `~/.hindsight/config.json`：`bank_id_template: hermes-{profile}`（私有流水）；cron「shared-bank 精选漏斗」(`hindsight_shared_funnel.py`) 每日精选入 shared；侧命令 `hmemh`（`~/.local/bin/hmemh`，默认 tag `agent:hermes-default`） |
| zcode | hooks 已装（SessionStart/UserPromptSubmit/Stop）；autoRetain → 私有 `zcode` bank；recall 自动读 shared；`save_shared.py` 手动写 shared（`agent:zcode`） |
| dsh | 私有流水：hooks（`hooks-claude-code`，每步用户输入 + 每次工具调用自动 retain → `dsh-<workspace>`）；共享：`hmem`（`~/.local/bin/hmem` → 本目录 `bin/hmem.sh`），手动按需读写 shared（`agent:dsh`） |

## DSH 侧（2026-11-05）：skill + 热记忆两层

DSH 没有 hermes 那样的自动记忆 provider，所以用两个互补的层：

| 层 | 位置 | 生效方式 | 放什么 |
|---|---|---|---|
| **热记忆（SOUL.md 等价物）** | `~/.dsh/AGENTS.md` 的 `## HOT MEMORY` 区块 | `dsh-agent-instructions` 每次会话自动注入上下文（默认开启，链预算 `maxBytes: 65536`；用户全局 `~/.dsh/AGENTS.md` + 项目 `AGENTS.md`/`CLAUDE.md` 链） | ≤25 行"现在仍然有效、每次都需要"的事实与铁律 |
| **语义记忆** | Hindsight `shared` bank | 按需 `search`（语义 + 图谱） | 结论级记忆：决策、根因、环境事实、偏好、可复用步骤 |

- skill：`~/.dsh/skills/shared-memory/`（`SKILL.md` + `scripts/smem.py` + `scripts/session_recall.py` + `references/upgrade-auto-recall.md`）
- CLI：`python ~/.dsh/skills/shared-memory/scripts/smem.py {ping|search|save|reflect|stats|recent}`（纯标准库，直连 mesh 地址；默认 tag `agent:dsh`，可用 `SMEM_AGENT` 覆盖）
- 约定：会话开始查一次记忆；有结论时写一条自包含的结论（含日期与原因）；若该结论"每次会话都需要"，同时写进 HOT MEMORY。
- 实测：save → 异步抽取 → search 命中闭环已验证（2026-11-05）。
- 想让 recall 真正**自动注入**（而不是靠 agent 自觉），用 `dsh-hooks-claude-code` 桥接 + `session_recall.py`，步骤见 skill 的 `references/upgrade-auto-recall.md`（需装插件并重启 DSH，本目录 `hooks/` 就是同一思路的旧实现）。

## dsh 侧全量私有流水（hooks，方案 1a）

- 触发：`@deepseek-ai/dsh-hooks-claude-code`（`~/.dsh/profiles/web/cordis.patch.yml` 的 `hooks-hindsight` insert），
  hooks.json 在 `hooks/hooks.json`，retain 脚本 `hooks/retain.py`（纯 python3 标准库 + 本地 REST）。
- 事件：`UserPromptSubmit`（每步新内容，即用户输入）→ `kind:user-prompt`；`PostToolUse`（工具名/入参/结果）→ `kind:tool-use`。
- 落点：私有 bank `dsh-<workspace-slug>`（如 `dsh-mnt-d-dshprojects`；`DSH_PRIVATE_BANK` 可覆盖），
  默认 tag `agent:dsh, ws:<slug>, sid:<前8位>`；type=`observation`；retain 走 async 管线。
- shared 逻辑不变：hooks 只写私有 bank；shared 仍只来自 hermes 精选漏斗 + zcode/dsh 手动写。
- 边界：1a 不含纯文本 assistant 回答（正文要等 transcript 读取，见 `三端记忆共享方案.md` 的 1b/旁路）。
- 状态：**已生效**（2026-09-03 重启后验证）。smoke：本会话日志出现成对 `hook/invoked`/`hook/result`；
  私有 bank `dsh-mnt-d-dshprojects` 自动创建，74 documents → 213 nodes；recall 可语义召回（tags 正确，
  GUI 与飞书会话均写入）。排查：会话日志 `hook/result` 事件或设 `DSH_HMEM_DEBUG=1`。
- 回滚：从 `cordis.patch.yml` 删除 `hooks-hindsight` insert（备份见 `.bak-*`）+ 重启。

## dsh 侧用法（hmem）

```bash
hmem bank                                   # 看服务/bank
hmem save "要记住的内容" [tag...]             # 写入 shared（默认 tag: agent:dsh；异步约 20~30s 后可查）
hmem search "查询" [tag...]                  # 语义检索 shared
hmem reflect "问题"                          # LLM 综合回答
hmem stats                                  # 库统计（完整 JSON，可管道解析）
```

- 环境变量：`HMEM_BASE` / `HMEM_BANK` / `HMEM_TYPES`（默认 `observation,world,experience`）
- 退出码：0 成功 / 1 用法错误 / 2 网络或连接失败 / 3 服务端错误（无 traceback，错误信息可读）
- **约定（用户确认 2026-09-02）**：DSH 不自动记录全量对话、不做抓取；只在**需要时**把
  值得跨端共享的内容（用户偏好、环境事实、技术决策、项目约定、踩坑结论）写入 shared。
  任务开始前先 `hmem search` 拉相关记忆。

## 验证记录

- [x] WSL↔Windows 双向读写 shared 打通；hermes client / zcode hooks recall / dsh hmem 均实测
- [x] hmem 加固（2026-09-02）：stats 完整 JSON、错误码规范、curl `--noproxy --connect-timeout` 防挂起
- [ ] zcode 真实会话 recall 注入验证（hooks 装后需重开 ZCode 实测一轮）
- [x] dsh hooks 私有流水 smoke（2026-09-03）：重启后本会话 hook/invoked+result 成对出现；
      bank 自动创建，74 docs→213 nodes；recall 语义召回 47 条，GUI+飞书会话均写入

## 回滚 / 维护

- hermes 私有流水：`~/.hindsight/config.json` 备份见 `config.json.bak.*`
- zcode hooks：还原 `C:\Users\风雨同行\.zcode\cli\config.json.bak.*` + 删 `~/.zcode/hooks/hindsight/`
- 精选漏斗 cron：hermes cron jobs.json「shared-bank 精选漏斗」条目
- bank 级清理：`DELETE /v1/default/banks/{bank_id}`（无单条删除；DELETE /memories 会清空全库）

## 备注（踩过的坑）

- bank 创建 `PUT /v1/default/banks/{bank_id}`；retain `POST /banks/{bank}/memories`；recall `POST /banks/{bank}/memories/recall`；reflect `POST /banks/{bank}/reflect`
- hindsight-zcode 官方安装器在 Windows 中文路径有 JSON 转义 bug → zcode hooks 为手工按官方布局安装（`python3` 命令改为绝对路径 python.exe）
- 本地服务无鉴权且监听 0.0.0.0，注意本机/局域网暴露边界
