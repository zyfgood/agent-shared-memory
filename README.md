# 三端共享记忆 · 实况（2026-09-02 复盘后）

dsh / zcode / hermes 三端共享记忆体系。文档为**当前实况**（2026-09-02 复盘核对），
与 `三端记忆共享方案.md`（初始落地记录）配合阅读。

## 架构：私有流水 + 精选共享

```
hermes 各 profile  ──自动每轮──▶ hermes-default / hermes-hermesatdeepseek / hermes-hermesforwiki / hermes（私有流水）
hermes 精选漏斗 cron ──每天 8:30──▶ shared（从 hermes-default 选近 26h 新增，GLM-5.3 同义去重，≤10 条/天，失败推运维群）
zcode             ──自动每轮──▶ zcode（私有流水，retainBankId=zcode）
zcode save_shared.py ──手动──▶ shared（agent:zcode）
dsh hooks         ──自动每步──▶ dsh-<workspace>（私有流水，UserPromptSubmit/PostToolUse）
dsh hmem save     ──手动按需──▶ shared（agent:dsh）
所有端 recall（hermes/zcode 自动、dsh 手动 hmem search）◀── shared（精选层）
```

要点：**shared 只放值得跨端共享的结论级内容**；每端全量流水留在各自私有 bank。

## 现状核对（2026-09-02）

| 项 | 值 |
|---|---|
| 记忆服务 | 本机 Hindsight：`http://127.0.0.1:9177`（WSL，hindsight-api + Postgres，无鉴权） |
| banks | `hermes-default`(578) · `zcode`(23) · `shared`(44) · `hermes-hermesatdeepseek`(93) · `hermes-hermesforwiki`(95) · `hermes`(18)（facts 数随运行变化） |
| hermes | `~/.hindsight/config.json`：`bank_id_template: hermes-{profile}`（私有流水）；cron「shared-bank 精选漏斗」(`hindsight_shared_funnel.py`) 每日精选入 shared；侧命令 `hmemh`（`~/.local/bin/hmemh`，默认 tag `agent:hermes-default`） |
| zcode | hooks 已装（SessionStart/UserPromptSubmit/Stop）；autoRetain → 私有 `zcode` bank；recall 自动读 shared；`save_shared.py` 手动写 shared（`agent:zcode`） |
| dsh | 私有流水：hooks（`hooks-claude-code`，每步用户输入 + 每次工具调用自动 retain → `dsh-<workspace>`）；共享：`hmem`（`~/.local/bin/hmem` → 本目录 `bin/hmem.sh`），手动按需读写 shared（`agent:dsh`） |

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
