# T002_PathProbe 优化报告

> **任务：** 对 T002_PathProbe 项目进行全面审查与优化，在保证现有功能稳定的前提下实施增强方案  
> **执行日期：** 2026-11-02  
> **执行者：** AG（ZCode）  
> **状态：** 已完成（执行范围经用户多轮确认，产品方向经 AC 审核并入 PRD Draft 0.2）

---

## 1. 执行摘要

原任务（诊断 → 方案 → 确认 → 执行）在中途按用户意图扩展为**产品化路线**：经头脑风暴、竞品扫描、AC（ChatGPT）审核，项目定位从「自用探测脚本」升级为「EnvDoctor — Windows 开发环境只读体检 CLI」，并建立 PRD 治理（Draft 0.2，已上传 GitHub）。本报告记录全部技术诊断、决策过程与代码执行结果。

**核心成果：**

- 主力脚本升级 V2.1 → V2.2：修复 4 类实测 bug，实现 PRD 首版能力（probe/check/diff 三动词、四级诊断、基础 diff、schema_version 数据标准）。
- 全量探测耗时从约 20 秒降至 3–6 秒；报告编码损坏问题根治。
- V1 脚本按约定废弃标注（不删除）；所有被改文件均有 .bak 备份。
- 过程中新发现的 2 个本机真实环境问题（PATH 含不可见字符、磁盘水位偏低）已被工具自己诊断出来——工具的有效性获得实证。

## 2. 诊断结论（原始审查，均有产出物实证）

| # | 问题 | 实证 |
|---|---|---|
| P0-1 | V1 未处理 wsl.exe 的 UTF-16LE 输出，NUL 字节写入报告 → 文件成非法 UTF-8 | `pathprobe_report.md` 被 file 判定为 data；V2 输出乱码「WSL Hr,g:」 |
| P1-2 | V1 无超时；V2 扩展探测（pip/npm/docker）无超时 | 代码审查 |
| P1-3 | Docker daemon 未运行时误报 0 容器 | V2 逻辑审查 + 本次实测对照 |
| P1-4 | PS 5.1 下 try/catch 对原生命令退出码无效 | 代码审查 |
| P1-5 | where.exe 首命中可能为存根/不可执行项（退出码 49） | V1 报告 python3 案例 |
| P2-6 | V1/V2 双脚本清单重复硬编码，无废弃标注 | 代码审查 |
| P2-7 | 输出路径硬编码 D:\0_M9_Kanglai\... | pathprobe.ps1 第 4 行等 |
| P2-8 | 输出与 00_Task.md 规格偏差（缺来源列/状态列、英文状态） | 报告对照 |
| P2-9 | 命名/版本号不一致（文件名 vs .EXAMPLE vs .NOTES） | 代码审查 |
| P3-10 | Start-Job 每命令一个子进程，15 命令串行约 20 秒 | V2 逻辑 + 实测 |

## 3. 决策过程（方案确认链）

1. **《项目诊断与增强方案》**：3 项轻量优化（L1 编码修复 / L2 配置统一与路径自适应 / L3 输出合规）+ 2 项功能增强（F1 快照 diff / F2 引擎加固）。
2. **头脑风暴（用户发起）**：产品定位讨论 → 「其他用户有没有需求」→ 竞品扫描（结论：「Windows 整机开发环境只读体检」位置空缺，winget configure 为声明式装机、缺反向审计）→ 产品形态（单文件 + probe/check/diff 三动词）→ 空环境用户问题（引入 Baseline/处方层）。
3. **与 WinMew（T013）撞车检查**：确认**独立产品、松耦合协同**（schema/哲学/叙事三对齐点，不合并）。
4. **PRD 治理**：按 WinMew PRD 格式起草 `EnvDoctor_PRD_Draft.md`（Draft 0.1）并上传 GitHub；AC 审核后迭代 Draft 0.2：**确认定位=专业诊断工具、首版收缩为「扫描+诊断+报告+基础对比」**（Baseline/处方后置）、**四级诊断替代健康分**、**JSON Schema 先行**、五条产品原则。
5. **执行范围**：发布门槛（L1/L2/L3）+ 首版核心（四级诊断、基础 diff、schema_version），即本次代码实施内容。

## 4. 执行记录

### 4.1 备份与废弃（修改前完成）

| 动作 | 文件 |
|---|---|
| 备份 | `02_Work\pathprobe.ps1` → `pathprobe.ps1.bak`（2551 字节一致） |
| 备份 | `02_Work\01_EnvProbe.ps1` → `01_EnvProbe.ps1.bak`（9158 字节一致） |
| 废弃标注 | pathprobe.ps1 文件头加【已废弃】说明（原因三条 + 指向 01_EnvProbe.ps1），原逻辑未删 |

### 4.2 V2.2 变更明细（01_EnvProbe.ps1 重写）

| 类别 | 变更 |
|---|---|
| 参数化 | 新增 `-Verb probe/check/diff`、`-With`、`-Compare`；`-OutputPath` 默认改为脚本相对路径自适应；`-Scope`、`-TimeoutSec` 保留 |
| 编码修复（L1） | 统一「原始字节捕获 + 编码嗅探」（BOM 优先 → UTF-16LE NUL 特征 → UTF-8）；报告/快照 UTF-8 无 BOM |
| 配置统一（L2） | 命令清单与版本参数（含 ffmpeg 单横线修正）、环境变量清单、敏感变量正则、诊断阈值全部集中脚本顶部配置区 |
| 输出合规（L3） | PATH 合并表（序号/路径/来源）、命令表补状态列（已安装/未安装/超时）、环境变量表补 PATH 行、状态中文化；JSON 字段保持英文稳定并新增 `schema_version=1` |
| 引擎加固（F2） | Start-Job → .NET `Process.Start` + 异步流读字节 + `WaitForExit(timeout)` 强杀；命中择优（.exe/.cmd 优先、WindowsApps 存根降级）；存根无输出自动尝试下一命中并记录证据；Docker 三态（not_installed/daemon_down/running）；pip/npm/docker 探测统一 30 秒超时 |
| 诊断（新增） | 四级诊断 R1–R6（详见提案文档），结论入报告与快照，check 动词终端彩色输出 + 退出码 0/1/2 |
| diff（F1） | 与历史快照基础对比（PATH/命令/环境变量的新增/移除/变更），支持 `-With` 指定旧快照或 `-Compare` 自动取上次 |

### 4.3 验证过程中发现并修复的 4 个次生问题（对复核有参考价值）

| # | 现象 | 根因 | 修复 |
|---|---|---|---|
| B1 | PS 5.1 解析报错，错误信息中文乱码 | 无 BOM 的 UTF-8 脚本被 PS 5.1 按 ANSI(GBK) 解析，中文字面量破坏语法 | 脚本文件加 UTF-8 BOM（输出文件仍无 BOM） |
| B2 | pip/npm 计数 -1、GPU 空、docker 误判未运行 | `Start-Process -PassThru` 在 PS 5.1 下 WaitForExit 后 ExitCode 仍为空（实测复现） | 改用 `[Diagnostics.Process]::Start` + 异步 `CopyToAsync` 读字节流，ExitCode 可靠且消除临时文件 |
| B3 | 报告混入孤立 CR（file 报 CRLF+CR） | 捕获的原生输出含 \r\n，截首行后残留 \r 进表格 | 捕获文本统一 normalize 换行 |
| B4 | 「不可见字符」诊断不触发 | LRE（U+202A）等格式字符在 CurrentCulture 排序中被视为可忽略，`-ne` 误判相等 | 改用 `[String]::Equals(..., Ordinal)` 序数比较 |

### 4.4 工具自证的诊断有效性（本次实测发现的本机真实问题）

1. 用户级 PATH 条目 `‪C:\tools\WindowsTerminal` 含不可见控制字符 U+202A（R2，info）。
2. D 盘可用空间 10% 触及水位线（R6，warning）。
3. git/npm/docker/curl/wsl 多版本共存提示（R4，info）——符合 AC「共存非问题」的判定原则。

## 5. 验证记录

| 验证项 | 结果 |
|---|---|
| probe（PS 5.1） | 通过，2.9–5.6 秒；15 命令逐项有结果；pip 49 / npm 0 / Docker 运行中 2 容器 2 镜像 / GPU GTX 1080 Ti 582.78，与独立对照一致 |
| probe（pwsh 7.6.9） | 通过，结果一致 |
| check | 通过；终端四级彩色输出；退出码=1（存在警告），语义正确 |
| diff | 通过；与 2026-09-12 V2.1 快照对比，PATH/命令/环境变量漂移完整呈现（Oracle Java 三件套移除、Python 3.14 迁移、ollama 卸载等） |
| 编码 | 报告/快照均为合法 UTF-8（file 校验通过），无孤立 CR，无 NUL 字节 |
| wsl 乱码 | 已修复：`WSL 版本: 3.0.3.0`（原「WSL Hr,g: 2.7.14.0」） |
| python3 | 本机实测确实无 python3，标「未安装」符合「不猜」原则 |
| 备份 | 两个 .bak 与原文件字节一致 |

> 注：验证输出写入 `03_Output\verify\`，未覆盖 `03_Output\` 下的历史报告（pathprobe_report.md 的损坏原件按约定保留作实证）。

## 6. 交付物清单

| 交付物 | 位置 |
|---|---|
| 本报告 | `T002_PathProbe\T002_PathProbe_优化报告.md`（本地）+ GitHub |
| 升级后主力脚本 | `T002_PathProbe\02_Work\01_EnvProbe.ps1`（V2.2） |
| 废弃标注的原型 | `T002_PathProbe\02_Work\pathprobe.ps1`（仅头部注释变更） |
| 备份 | `02_Work\*.ps1.bak` × 2 |
| PRD | `02_Work\PRD\EnvDoctor_PRD_Draft.md` Draft 0.2 + [GitHub](https://github.com/molkr/MolKr-Work/blob/main/0_M9_Kanglai/02_Task/T002_PathProbe/02_Work/PRD/EnvDoctor_PRD_Draft.md) |
| 诊断规则与数据标准提案 | `02_Work\Design\EnvDoctor_V1_诊断规则与数据标准_提案.md`（AC 步骤 4 产出物）+ GitHub |
| 验证产物 | `03_Output\verify\`（report + snapshot） |

## 7. 遗留与后续

1. **待 AC 复核**：诊断规则细则、JSON Schema v1、验收标准（提案文档 §5 列出 5 个复核点）。
2. **待用户确认**：项目命名（需查重）、报告语言策略、快照留存策略（覆盖式 vs 日期留档）、代码仓库位置（Molkr-Work 子目录 vs 独立仓库）——对应 PRD §9。
3. **代码未入库**：脚本与 .bak 仅在本地，因「仓库位置」未定；确认后随下一提交入库。
4. **后续版本**（按 PRD 优先级）：Baseline 与装机/迁移场景、处方层、AI 工作站专项、PSGallery 分发。
