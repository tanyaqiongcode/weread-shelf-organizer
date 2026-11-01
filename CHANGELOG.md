# Changelog

本项目遵循 [Keep a Changelog](https://keepachangelog.com/) 规范，版本号遵循语义化版本。

## [1.3.0] - 2026-11-01

Windows / 跨平台派生版本，基于上游 `main` 的 P1/P2 安全加固。

### 新增
- **`WEREAD_COOKIE` 凭据**: 支持 `wr_vid` / `wr_skey` Cookie 字符串（允许引号与 `=`），
  保留 `WEREAD_VID` / `WEREAD_SKEY` 与 `WEREAD_LOG_DIR` 兼容
- **Windows 平台提示**: 非 macOS 凭据失败时给出 Cookie/环境变量指引；Windows 控制台输出
  使用 UTF-8 容错，避免中文/emoji 编码异常
- **CI**: 单测矩阵加入 `windows-latest`，与 Ubuntu 同时执行

### 安全
- **删除分组前二次核验**: `groups --delete` / `--purge-empty` 不再信任启动时的快照,
  每次删除前重新同步并核对 archiveId + 分组名 (+ 是否仍为空); 期间加入书籍、分组改名/
  删除/重建时跳过删除。`--force` 允许非空组删除, 但仍核对 archiveId/名称。

### 修复
- **中断报告保留已完成批次**: 凭据失效时 apply/review 的报告此前 moved=0、全部记为 failed;
  现 `_do_moves` 就地追加调用方传入的进度列表, 报告如实反映已完成批次
- **报告原子写入 + 逐批检查点**: `--json-report` 改为同目录临时文件 + `os.replace`,
  每批结束即落盘; 进程中断不会留下半截 JSON, 已完成进度可追溯
- **非 401 中断同样保留进度**: Ctrl-C/意外异常先写 `interrupted` 报告再原样抛出;
  核验阶段异常也会落盘已完成的迁移结果
- **跨平台凭据**: 非 macOS 无 `open` 命令不再抛 FileNotFoundError; 环境变量凭据校验遇
  URLError/超时也不再 traceback, 统一以 FATAL + 退出码 2 结束
- `apply --dry-run` 不再获取凭据/启动客户端; `groups --list` 帮助文本与行为对齐

### 工程
- 单测矩阵覆盖 Python 3.9 / 3.11 / 3.13; 测试 44 → 62 项（新增 Cookie 解析与 Windows 提示）
- 新增 `tests/fixtures/` 脱敏接口样例（sync/archive/deleteArchive/非 JSON），
  覆盖删除前复核、原子报告、逐批检查点与非 401 中断分支
- README pip 安装说明补充示例规则下载; AGENTS.md 去除不存在的 scripts/plan.json、
  同步测试数与新增安全说明

## [1.2.1] - 2026-09-06

### 安全
- **review 入口与 apply 同等保护**: 移动「来自已有分组」的书必须 --yes(退出码 3),
  双入口共用确认校验, 消除保护差
- **中断进度报告**: 凭据失效不再深处直接退出——抛 CredentialsExpiredError,
  apply/review 捕获后把未尝试书籍计入 failed、写含 interrupted 标记的报告,
  以退出码 2 结束; 重新登录后按报告精确续传
- **review 未知目标分组**计入 failed 并非零退出(此前静默成功)

### 工程
- CI 固定 ruff@0.12.0 / mypy@1.17.1, 恢复 format 门禁; 测试 33 → 35 项
- README 测试数更正、商用表述与 MIT 许可对齐; SECURITY 支持表升至 1.2.x

## [1.2.0] - 2026-09-06

### 新增
- **网络层指数退避**: 429/5xx/网络超时按指数退避+抖动自动重试, 遵循 Retry-After;
  其他 4xx 快速失败 (http_json 层, 全命令受益)
- **审计标识**: plan 生成 plan_id 与 shelf_hash(计划时点归组状态哈希);
  apply/review 每次执行生成 batch_id, 报告新增 batches 批次摘要(失败批次含 API 响应错误摘要)
- **pyproject 包元数据**: 可 pip 安装/升级, 提供 `weread-shelf` CLI 入口
  (requires-python >=3.9, venv 实测通过)
- **安全迁移演练文档**: docs/walkthrough.md 完整展示
  plan --dry-run → 人工审阅 → apply --yes → verify 全流程

### 变更
- 批量迁移返回值增加批次摘要 (_do_moves → 三元组)

## [1.1.1] - 2026-09-02

### 修复
- 系统分组「归档」(archiveId=1) 视同未分组——客户端会把未分组书放入该容器承载，
  此前执行前状态校验会误跳过这部分书的迁移；附 2 项回归测试
- 示例规则: 精品小说-社会小说 改向 09 类型小说（原 →08 世界文学，
  会把陈彦、豆豆等中国当代社会小说误导向外国文学）

### 补充
- 示例规则补充 8 条实战高频映射（漫画类、计算机-人工智能、励志/人生哲学等）
- api.md 补充归档容器行为说明与 shelf/add、shelf/delete 接口参考
- 仓库文档措辞统一：「全量重组」→「书架重组（白名单确认）」

## [1.1.0] - 2026-08-30

### 安全
- 重组模式做实：`plan --reorganize --source-group "白名单"` 只迁移白名单来源分组内的书，
  计划记录每本书的来源（`from`）与来源分组快照（`source_snapshot`）；
  含重组迁移的 `apply` 必须加 `--yes` 人工确认（退出码 3）
- `apply`/`review` 执行前重新同步书架，逐本校验「当前分组 == 计划来源分组」，
  状态不符自动跳过——过期或被篡改的计划无法绕过安全线

### 修复
- `classify()` 返回值元数不一致导致 plan 在首本待复核书处崩溃（v1.0.0 严重缺陷）
- `apply`/`review` 存在迁移失败或核验错位时以非零退出码结束（此前误报成功）
- 脚本日志脱敏 vid；裸 `open()` 全部改为上下文管理器

### 新增
- `--json-report`：apply/review 输出结构化 moved/failed/stale/wrong 报告，供定时任务判断成败
- 置信度按匹配类型取值（完整命中/主栏目命中可配），`min_confidence` 阈值生效，
  低于阈值的进复核清单并附候选建议
- verify 升级为校验「预期分组」而非「任意目标分组」
- 环境变量凭据路径 `WEREAD_VID`/`WEREAD_SKEY`（Linux/Windows/CI 可用）
- 26 项离线回归测试 + GitHub Actions CI

## [1.0.0] - 2026-08-28

- 首次发布：全量重组、增量归类、分组创建/清理、迁移核验
- 12 分组示例规则、接口参考文档、演示动图
