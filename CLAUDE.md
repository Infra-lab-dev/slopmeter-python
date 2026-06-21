# CLAUDE.md — slopmeter 开发指南

本文件给在此仓库工作的 AI/开发者快速上手用。本仓库是 [KMnO4-zx/slopmeter-python](https://github.com/KMnO4-zx/slopmeter-python)(MIT)的个人 fork,远程为 `origin` = `Infra-lab-dev/slopmeter-python`(分支 `main`)。

## 项目是什么
一个**本地**工具:扫描各 AI 编码工具(Claude Code / Codex / Cursor / Gemini / opencode 等)写在本机的使用日志,生成 **GitHub 贡献墙式的 token 使用热力图**,提供命令行导出和本地网页查看器。100% 本地、不外发。

## 本 fork 在原版基础上新增的功能
1. **终端 `slopmeter day [YYYY-MM-DD]`**(默认今天):打印某天明细——总量/输入/输出/缓存、各模型表(含每模型成本)、以及 **0–23 小时火花图 + 最忙时段**。
2. **网页"某天明细面板"**:点击热力图任意格子 或 用右上角日期框 → 弹出固定面板,含完整模型表、每模型成本、**彩色 0–23 小时柱状图(带时间轴)**。
3. **单工具时自动隐藏聚合(Aggregate/"all")块**:主页卡片和某天面板都生效(只有 1 个工具有数据时,聚合等于它本身,冗余)。

## 开发命令(用 uv)
```bash
# 从源码运行(改完代码即时生效,推荐开发时用)
uv run slopmeter day 2026-06-21 --claude     # 终端某天明细
uv run slopmeter serve --port 8770           # 本地网页(开发验证)
uv run slopmeter export --claude --format json -o /tmp/out.json  # 导出 JSON 查数据

# 跑测试(pytest 不在默认依赖里,用 --with 临时带上)
uv run --with pytest pytest -q

# 把改动装进全局 `slopmeter` 命令(uv tool 是拷贝源码,改完必须重装才生效)
uv tool install . --force
```
> ⚠️ `uv tool install` 把源码拷进独立环境。**编辑源码后,全局 `slopmeter` 不会自动更新**,必须重跑 `uv tool install . --force`。开发期间用 `uv run` 避免混淆。

## 架构 / 关键文件(`src/slopmeter/`)
- `cli.py` — Typer 命令(`serve` / `export` / `day` + 默认 callback)。核心:
  - `analyze_usage(values, *, selection_mode, window=None)` — 扫描+聚合,返回 `AnalysisBundle`。`window` 可传自定义时间窗(`day` 命令传"目标日 00:00–23:59")。
  - `day_command` / `run_day` / `render_day_report` — 终端某天明细(含小时火花图)。
  - `get_default_service_provider_ids` — serve 默认选哪些 provider;**单工具时不选 "all"**(隐藏聚合)。
- `providers/*.py` — 每个工具一个 loader,读各自日志、解析时间戳+token。`claude.py` 读 `~/.claude/projects/**/*.jsonl`(有精确时间戳)。
- `utils.py` — 聚合核心。`DailyTotalsByDate` / `TokenTotalsByDate`(含 `hourly[24]`)、`add_daily_token_totals`(按真实小时累加)、`totals_to_rows`、`merge_daily_totals_by_date`、`merge_usage_summaries`(聚合也合并 hourly)。
- `models.py` — 数据类。`DailyUsage`(含 `hourly: list[int]` 长度 24)、`TokenTotals`、`ModelUsage`、`UsageSummary`。
- `pricing.py` / `model_prices.py` — 计价。`compute_daily_cost` / `compute_model_usage_cost`。
- `export.py` — 序列化成 JSON payload。`daily_usage_to_dict`(含 `hourly`、`costUsd`)、`model_usage_to_dict`(含每模型 `costUsd`)。
- `render.py` — **整个网页(HTML+CSS+内联 JS)是一个超大 f-string**(见 `render_html_document`)。网页某天面板逻辑在 JS 函数 `renderDayDetail` / `selectDay`;小时柱状图在其中生成。
- `server.py` — serve 模式的本地 HTTP 服务 + PNG 导出端点。

## 数据流(理解后改动才不会错)
日志条目 → 各 provider loader → `add_daily_token_totals`(按日期聚合,**有 datetime 时按小时累加到 `hourly[hour]`**)→ `totals_to_rows` 生成 `DailyUsage[]` → `UsageSummary` → `export.py` 序列化 → 终端打印 / 网页 payload。聚合("all")由 `merge_usage_summaries` 合并各 provider(也合并 hourly)。

## 重要约定 / 踩坑点
- **render.py 的 JS 在 f-string 里**:所有字面大括号必须写成 `{{` `}}`,模板插值 `${expr}` 要写成 `${{expr}}`。改 JS 极易因此出错;改完务必跑一次 `uv run slopmeter serve` 或 `export --format html`(能成功生成 = f-string 完整)。
- **小时数据的限制(故意为之)**:小时分布只对**带精确时间戳**的日志准确(Claude JSONL 有)。`claude.py` 的 stats-cache 回退只精确到天,故意传 `timestamp.date()` 给 `add_daily_token_totals`,**避免伪造一个 0 点尖峰**。只来自回退源的那天会显示"无小时数据"。校验不变量:`sum(hourly) == day.total`。
- **验证网页**:`uv run slopmeter serve` 起服务(它在启动那一刻扫一遍日志生成快照,新用量需重启才出现),再用浏览器打开;file:// 被禁,必须走 http://127.0.0.1。
- **测试风格**:`tests/test_cli.py` 用 Typer `CliRunner` + `base_env(tmp_path, CLAUDE_CONFIG_DIR=...)` 写临时 jsonl 日志来驱动。新增行为要补对应测试(当前全绿)。

## Git / 上传
- 远程已配好(`origin`,HTTPS)。该机器家庭网络会干扰 github.com 的 HTTP/2 → 已全局设 `git config --global http.version HTTP/1.1`;若 push 仍失败,切手机热点。
- 凭据走 `gh`(`gh auth setup-git` 已配)。当前登录账号 `Infra-lab-dev`。
- 原项目的 `.github/workflows/publish.yml`(发布到 PyPI 的 CI)已删除——当前 gh token 无 `workflow` 权限,保留会导致 push 被拒。若要 CI 需 `gh auth refresh -s workflow` 后再加回。

## 待办 / 可扩展点
- 小时分布目前粒度=每小时总 token;如需按输入/输出拆分,要扩 `hourly` 的结构(目前是 `list[int]`)。
- 多工具(如同时有 Codex/Cursor 数据)场景下聚合卡才会显示,届时小时图也会叠加多工具——逻辑已就绪但本机暂只有 Claude 数据,未实测多工具。
