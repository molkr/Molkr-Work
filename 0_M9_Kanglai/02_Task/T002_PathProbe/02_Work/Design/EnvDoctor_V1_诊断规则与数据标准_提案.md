# EnvDoctor — V1 诊断规则与数据标准提案

> **文档类型：** 技术提案（AC 审核推进步骤 4 的落地物）  
> **当前状态：** 待 AC 复核  
> **重要说明：** 本文对应 PRD Draft 0.2 §9 待确认问题 5/6/7（诊断规则细则、JSON Schema 定稿、首版验收标准）。所列规则 V1 已全部在 `02_Work\01_EnvProbe.ps1`（V2.2）中实现并通过实测，本文将其文档化并提交复核；复核意见将反哺代码迭代。

| 项目 | 当前信息 |
|---|---|
| 项目名称 | EnvDoctor（暂定名） |
| 任务编号 | T002 |
| 文档版本 | Propose 0.1 |
| 文档维护 | AG 起草，AC 与用户复核 |
| 更新日期 | 2026-11-02 |

---

## 1. 四级诊断等级定义

| 等级 | 代码 | 含义 | 退出码影响 |
|---|---|---|---|
| 正常 | （无条目） | 未发现明显问题 | 0 |
| 提示 | info | 存在值得注意的情况，通常无需处理 | 不影响 |
| 警告 | warning | 可能影响使用，建议关注或处理 | 1 |
| 异常 | critical | 明确存在故障或配置错误 | 2 |

**升级需证据原则（PRD §6.4 已确认）：** 现象本身（如多版本共存）不构成升级理由；仅当获得冲突证据（存根遮蔽、解析失败、连接失败等）时才提升等级。

## 2. V1 诊断规则清单

| ID | 名称 | 触发条件 | 等级 | 证据与升级条件 | 建议文本方向 | 实现状态 |
|---|---|---|---|---|---|---|
| R1 | PATH 失效条目 | PATH 条目（去除引号/空白/不可见字符后）经 `Test-Path -LiteralPath` 判定不存在 | warning | 目录真实不存在即触发 | 确认卸载残留后手动清理 | 已实现 |
| R2 | PATH 条目含不可见字符 | 条目含 U+200B–200F / U+202A–202E / U+2060 / U+FEFF 等格式字符 | info | 发现即触发；若同时导致解析失败，由 R1 以 warning 承接 | 重新录入该条目 | 已实现 |
| R3 | 命令被存根遮蔽 | where.exe 首命中执行无输出且退出码非 0，而后续命中可正常输出版本 | warning | 双重证据：首选不可用 + 替代可用 | 调整 PATH 顺序或移除存根 | 已实现 |
| R4 | 命令多版本共存 | where.exe 返回 ≥2 个不同命中 | info | **不因共存升级**；升级仅凭 R3 类冲突证据 | 如需固定版本调整 PATH 顺序 | 已实现 |
| R5 | Docker daemon 未运行 | docker CLI 已安装且 `docker version` 退出码非 0 | warning | 连接失败即触发；与「未安装」严格区分 | 启动 Docker Desktop 后重测 | 已实现 |
| R6 | 磁盘水位 | 可用比例 < 10% → warning；< 5% → critical | warning / critical | 按比例分级，阈值集中于配置区 | 关注/尽快清理 | 已实现 |

技术实现备注：

- R2 的字符对比必须使用**序数比较**（`[String]::Equals(..., Ordinal)`）——CurrentCulture 排序将 LRE 等格式字符视为可忽略，`-ne` 会误判相等（实测复现）。
- R1 使用 `Test-Path -LiteralPath` 规避通配符歧义（PowerShell issue #21112）。
- 命令命中择优顺序：非 WindowsApps 的 .exe → .cmd/.bat → WindowsApps 存根（避免拉起商店）；存根探测采用「原始字节捕获 + NUL 特征 UTF-16 解码」，wsl.exe 中文输出已实证修复。

## 3. JSON 快照 Schema v1（数据标准）

设计约定：**只加不改**——后续版本仅允许新增字段或枚举值，不允许改名/改义/删除；`schema_version` 递增。Markdown 报告为渲染层，任何报告格式调整不影响本 schema。该 schema 同时是与 WinMew（T013）松耦合对齐的接口（WinMew 未来可直接消费快照文件）。

| 字段路径 | 类型 | 说明 |
|---|---|---|
| `schema_version` | string | 数据结构版本，当前 "1" |
| `product` | string | 固定 "EnvDoctor" |
| `timestamp` | string | 探测时间 `yyyy-MM-dd HH:mm:ss`（本地时区） |
| `script_version` | string | 脚本版本，当前 "V2.2" |
| `verb` | string | probe / check / diff |
| `hostname` / `user` | string | 主机与当前用户 |
| `path.user[]` / `path.system[]` | string[] | 用户级/系统级 PATH，保持注册表顺序，去除空项 |
| `commands[].name` | string | 命令名 |
| `commands[].path` | string | 实际生效路径（未安装为空） |
| `commands[].version` | string | 版本输出首行（原文，不做二次解析） |
| `commands[].status` | enum | installed / missing / timeout |
| `commands[].note` | string | 探测备注（如存根改用替代命中），无则空 |
| `envvars[].name` / `.value` / `.status` | string / enum | 敏感变量 value 固定 "(不记录)"；status: set / unset |
| `packages.pip_count` / `.npm_global_count` | int | 包数量，不可用为 -1 |
| `docker.state` | enum | not_installed / daemon_down / running |
| `docker.running` / `.all` / `.images` | int | 容器与镜像计数，不可用为 -1 |
| `disk[].name` / `.used` / `.free` / `.total` | string / number | 单位 GB，保留两位 |
| `gpu[]` | object | name / driver / memory_total / memory_used / memory_free（nvidia-smi 原文） |
| `diagnostics[].level` | enum | info / warning / critical |
| `diagnostics[].item` / `.detail` / `.suggestion` | string | 结论项 / 证据描述 / 建议 |

diff 动词附加字段：`diff_count`（int，差异条数）；差异明细仅入报告不入快照（保持快照为「时点状态」语义）。

## 4. 首版验收标准

### 4.1 功能验收（继承 T002 00_Task.md 并扩展）

1. 15 个命令逐项有结果，未安装标「未安装」，不猜测、不编造（T002 原验收项）。
2. PATH 清单完整、含来源列、序号连续（T002 原验收项）。
3. 环境变量逐项有结果，密钥类只记存在性不记值（T002 原验收项，已实测 DEEPSEEK_API_KEY 未记值）。
4. wsl 等 UTF-16 输出命令的版本行中文正常，报告为合法 UTF-8（修复项，实测 `WSL 版本: 3.0.3.0`）。
5. 全部探测有超时保护，超时标「超时」不挂死（实现项）。
6. Docker 三态正确区分（实测：daemon 运行中 → 2 容器/2 镜像，与独立对照一致）。
7. diff 与历史快照对比输出新增/移除/变更三类差异（实测跨两个月快照）。
8. check 退出码：0 无警告异常 / 1 有警告 / 2 有异常（实测本机=1）。
9. 输出路径自适应：无 -OutputPath 时默认脚本相对 `..\03_Output\`。

### 4.2 质量验收

1. Windows PowerShell 5.1 与 PowerShell 7 双版本运行结果一致（均实测）。
2. 报告与快照为 UTF-8 无 BOM；报告无孤立 CR（`file` 校验通过）。
3. 性能：全量探测（含扩展）耗时秒级，目标 < 10 秒（实测 2.9–5.6 秒，原 Start-Job 方案约 20 秒）。
4. 脚本文件本身带 UTF-8 BOM（PS 5.1 解析中文必需，实测教训）。

## 5. 待 AC 复核点

1. R6 磁盘阈值（10% / 5%）是否合理，是否需要按盘符类型（系统盘/数据盘）区分。
2. R4 多版本共存默认 info 的粒度——是否需要在「首命中与次命中主版本号不同」时升级。
3. Schema v1 字段是否有遗漏（尤其面向未来 AI Agent 消费场景的预留字段）。
4. 退出码语义（0/1/2）是否满足 CI/AG 调用约定。
5. 扩展探测超时固定 30 秒是否开放为参数。

## 6. 版本与变更记录

| 版本 | 日期 | 变更内容 | 状态 |
|---|---|---|---|
| Propose 0.1 | 2026-11-02 | 建立 V1 诊断规则清单（R1–R6）、JSON Schema v1、首版验收标准；全部规则已在 V2.2 实现并实测 | 待 AC 复核 |
