# weread-shelf-organizer-windows

微信读书书架自动整理 Skill — Windows / 跨平台版（Cookie 凭据）。

[![tests](https://github.com/tanyaqiongcode/weread-shelf-organizer/actions/workflows/test.yml/badge.svg?branch=windows)](https://github.com/tanyaqiongcode/weread-shelf-organizer/actions/workflows/test.yml)

> 这是 Mac 版 [weread-shelf-organizer `main` 分支](https://github.com/tanyaqiongcode/weread-shelf-organizer/tree/main)
> 的 Windows 派生分支。Windows 上通过浏览器 Cookie 提供凭据，**不依赖微信读书 Mac 客户端**；
> 需要 Mac 自动提取客户端日志，请使用 `main` 分支。

一个 [ZCode](https://zcode.ai) / Claude Code 风格的 Agent Skill：通过 `WEREAD_COOKIE`
（或 `WEREAD_VID` / `WEREAD_SKEY`）提供登录凭据调用书架接口，实现**分组的自动创建、
书籍的批量归类、迁移后的自动核验**。

![demo](docs/demo.gif)

*真实演示：新书《鞋狗》入库 → 自动识别分类 → 归入「03 商业经济」→ 核验落位*

## ✨ 能力

- **书架重组** — 按分类表把指定的旧分组整体迁入新体系（显式 `--source-group` 白名单 + `apply --yes` 双重确认，白名单之外的分组永不触碰）
- **从零分类** — 只给分组设想，AI 按平台分类 + 书名/作者逐本定类
- **增量整理** — 日常运行只处理新书/未分组书，适合每周定时任务
- **分组管理** — 自动创建缺失分组、清理迁移后清空的旧分组
- **安全核验** — 每批迁移后重新同步书架核验落位，报告未落位清单

## 📦 效果

真实书架 2924 本书、12 个目标分组的一次性重组：

| 分组 | 数量 | 分组 | 数量 |
|---|---|---|---|
| 01 成长学习 | 90 | 07 中文文学 | 320 |
| 02 语言写作 | 107 | 08 世界文学 | 564 |
| 03 商业经济 | 202 | 09 类型小说 | 617 |
| 04 历史社会 | 296 | 10 性别家庭 | 262 |
| 05 科技未来 | 84 | 11 人物传记 | 115 |
| 06 心智精神 | 242 | 12 艺术生活 | 25 |

2888 本一次性迁移，0 失败，全程约 4 分钟。

## 🚀 快速开始

### 1. 前置条件

- **Windows 10/11**（也可在 Linux / macOS 使用 Cookie 方式）
- Python 3.9+（仅标准库，无第三方依赖）
- 已登录微信读书网页版（用于获取 `wr_vid` / `wr_skey`）
- ZCode / Claude Code 等 Agent 环境，或直接手动调用脚本
- **不需要** 微信读书 Mac 客户端

### 2. 安装为 Skill

```powershell
# ZCode（Windows PowerShell）
git clone -b windows https://github.com/tanyaqiongcode/weread-shelf-organizer.git "$env:USERPROFILE\.agents\skills\weread-shelf-organizer-windows"

# Claude Code（Windows PowerShell）
git clone -b windows https://github.com/tanyaqiongcode/weread-shelf-organizer.git "$env:USERPROFILE\.claude\skills\weread-shelf-organizer-windows"
```

然后对 AI 说：

> 帮我把微信读书书架按这份表格整理一下 / 帮我整理微信读书书架的新书

### 3. 凭据（Windows 必读）

1. 浏览器打开 <https://weread.qq.com> 并扫码登录。
2. 按 F12 → Application / 存储 → Cookies → `https://weread.qq.com`，复制 `wr_vid` 与 `wr_skey`。
3. 在当前 PowerShell 会话设置 Cookie 并验证：

```powershell
$env:WEREAD_COOKIE = 'wr_vid=你的vid; wr_skey=你的skey'
python scripts\weread_shelf.py creds
```

也可分开设置 `$env:WEREAD_VID` / `$env:WEREAD_SKEY`（可选 `$env:WEREAD_V`）。

- `wr_skey` 是会话密钥，会过期；遇到 HTTP 401 重新复制即可。
- 环境变量仅对当前窗口有效，新开窗口需重新设置。**不要把 Cookie 写入仓库或截图分享。**
- 可选兜底：设置 `WEREAD_LOG_DIR` 指向微信读书 Windows 客户端日志目录，复用日志提取逻辑。
  该路径尚未在 Windows 实机验证，不作为默认方式。

### 4. 手动使用（不用 Agent 也行）

```powershell
# 方式 A: pip 安装（提供 weread-shelf 全局命令, requires-python >= 3.9）
pip install "git+https://github.com/tanyaqiongcode/weread-shelf-organizer.git@windows"

# pip 安装只包含 CLI, 不含 examples/; 先下载示例规则再按需修改
curl.exe -LO https://raw.githubusercontent.com/tanyaqiongcode/weread-shelf-organizer/windows/examples/rules.example.json

weread-shelf creds
weread-shelf plan --rules rules.example.json -o plan.json --dry-run
weread-shelf apply --plan plan.json
```

```powershell
# 方式 B: 克隆后在仓库根目录运行
python scripts\weread_shelf.py creds                    # 验证 Cookie/环境变量
python scripts\weread_shelf.py plan --rules examples\rules.example.json -o plan.json --dry-run
python scripts\weread_shelf.py apply --plan plan.json   # 执行迁移 + 核验
python scripts\weread_shelf.py groups --purge-empty     # 清理空分组
```

**把指定旧分组打散重排（重组模式，双重确认）：**

```powershell
# ① 基线 CSV + 白名单来源分组生成计划（只动白名单里的分组）
python scripts\weread_shelf.py plan --rules examples\rules.example.json --baseline my_base.csv --source-group "探照灯好书小说2,豆瓣评分9.0以上的书籍" --reorganize -o plan.json --dry-run
# ② 人工复核 plan.json 后确认执行（缺 --yes 会拒绝）
python scripts\weread_shelf.py apply --plan plan.json --yes --json-report report.json
```

分组规则见 `examples/rules.example.json`（12 分组示例 + 平台分类映射，改成分你自己的即可）。

## 🧠 工作原理

1. **凭据**：从 `WEREAD_COOKIE`（`wr_vid` / `wr_skey`）或 `WEREAD_VID` / `WEREAD_SKEY`
   读取，只驻留内存，不落盘不回显；如设置 `WEREAD_LOG_DIR`，可尝试从客户端日志提取。
   Mac 自动提取客户端日志请使用 `main` 分支。
2. **接口**：调用客户端同款官方接口 —— `/shelf/sync` 同步、`/shelf/archive` 建组/移书、
   `/shelf/deleteArchive` 删分组。明细见 [references/api.md](references/api.md)。
3. **分类**：平台分类（如"经济理财-商业"）按规则映射到分组（完整命中置信度 0.95、
   主栏目命中 0.75，阈值可配）；映射不到或低于阈值的交给 AI 按书名/作者判断，
   再拿不准的保留"需复核"清单由你定夺。
4. **核验**：迁移后重新同步书架，逐本确认落在**预期分组**（而非任意目标分组）；
   任何错位都会显式报告。

## 🧪 测试

```powershell
python -m unittest discover -s tests -v
```

63 个离线回归测试（全 mock、无网络，有状态假书架模拟迁移后同步）：classify 元数一致性、
含待复核书的端到端 plan、置信度阈值分流、正确分组核验、重组白名单边界、执行前状态校验、
核验失败非零退出、`--json-report` 结构化报告、环境变量凭据、`WEREAD_COOKIE` 解析、
批次切分、基线解析、删除前的 archiveId/空组二次核验、逐批原子检查点、非 401 中断报告、
脱敏接口样例等。推送后由 GitHub Actions 在 Ubuntu + Windows 自动执行。

`tests/fixtures/` 的脱敏响应样例用于锁定已知接口字段与安全分支；**它不能单独发现上游
接口变化**，真实变化仍依赖每周 `sync` 与迁移后 `verify` 的显式报告。

完整安全重组流程见 [docs/walkthrough.md](docs/walkthrough.md)。

**自动化集成**：`apply`/`review` 的退出码 — `0` 成功；`1` 存在迁移失败或核验错位；
`2` 凭据失效；`3` 重组迁移未加 `--yes`。配合 `--json-report` 可获得结构化的
moved/failed/stale/wrong 结果；报告每批原子更新，并带 `in_progress` / `interrupted` /
`error` 字段，进程中断或核验异常时仍保留已完成进度，便于续作与审计。

## 🛡️ 安全设计

- 默认只移动「未分组」的书 — 你手动放入其他分组的书永远不会被碰；重组模式只动你显式指定的来源分组，且必须 `--yes` 确认
- **执行前强制重校验** — `apply`/`review` 重新同步书架，逐本核对当前分组与计划一致，状态不符自动跳过（过期/被篡改的计划无法越线）
- 不删除书籍、不移出书架、不改变私密状态、不动专辑与文章收藏
- 删除分组只删空分组，且 `removeBooks: 0`（书永远保留在书架）；删除前重新同步核对 archiveId/名称/是否仍为空，快照过期或期间加入书籍则跳过
- 先 `--dry-run` 出计划再执行；每批 40 本、失败重试、迁移后核验

## ⚠️ 免责声明

本项目调用微信读书客户端内部接口（非官方开放 API），接口可能随版本更新而变化。
本项目按 [MIT](LICENSE) 许可开源（允许商用）；但接口非官方、账号风险自担，
严禁用于批量注册、爬取、代练等滥用场景。使用产生的任何后果由使用者自行承担。

## 来源

派生自 [tanyaqiongcode/weread-shelf-organizer](https://github.com/tanyaqiongcode/weread-shelf-organizer/tree/main)
（MIT）。`windows` 分支仅调整 Windows 凭据入口、平台提示与文档，迁移逻辑与安全红线保持一致。

## License

[MIT](LICENSE)
