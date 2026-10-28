# jam_ghost_deploy — 开发计划

> 状态：阶段一（研究 + 规划）已完成，业务代码尚未开始。完成一项即更新勾选状态。

## 1. WPKG 功能分析摘要（源码依据）

详见 `FEASIBILITY.md` §3。关键结论：

- 官方仓库 `github.com/wpkg/wpkg-js`（GPL-2.0），提示词给的 `sonicnkt/wpkg` 不存在（404）。
- 架构：XML 驱动（packages/profiles/hosts + config），主引擎 `wpkg.js`（JScript）。
- 安装状态检测：`<check type=registry|file|uninstall|execute|logical condition=exists|versiongreaterorequal|exitcodeequalto|not>`。
- 命令分离：install/upgrade/remove/downgrade + `<exit code= reboot=>` 白名单。
- 依赖：`<depends>` + `priority`。
- 本地/远程：`<download url= target=>`，`%SOFTWARE%` 网络共享，非公网下载。
- 日志：`logLevel` 位掩码、`logfilePattern`。
- **无 GUI 自动化**（全仓库无窗口/控件自动化代码，见 FEASIBILITY.md §3.2 第 10 点）。

## 2. 功能对照表

| 能力 | WPKG | 本项目需要 | 复用设计思路 |
|---|---|---|---|
| 清单驱动 | XML packages | YAML 清单 | 复用「清单定义一切」 |
| 安装状态检测 | check（registry/file/uninstall/execute/logical） | 同，另加哈希/签名验证 | 复用 check 类型与 condition 语义 |
| 命令分离 | install/upgrade/remove/downgrade | install/upgrade/remove + 静默/GUI 两级 | 复用命令分离 |
| 退出码 | exit 白名单 | 同 | 复用白名单思路 |
| 依赖/顺序 | depends + priority | 依赖图 + 拓扑排序 | 复用 depends 思想 |
| 本地/远程 | SMB 共享 + download | 本地目录优先 + 公网下载 + 校验 | 借鉴 download，改为本地优先 |
| 日志 | 位掩码日志 | logging 分级 + 文件/UI | 复用分级日志 |
| GUI 自动化 | 无 | pywinauto + PyAutoGUI 回退 | 独立实现 |
| 图形界面 | 无（CLI/服务） | PySide6 多选清单 UI | 独立实现 |
| 进度/取消 | 仅 sendStatus STDOUT | 实时进度 + 安全取消 | 借鉴状态上报，独立实现 |

## 3. 需求清单

- P0：本地安装包优先；静默优先；批量安装；日志；失败重试；结果验证；YAML 清单；安全（不绕过 UAC/不自动重启/不记录敏感）。
- P1：GUI 界面；进度/取消；依赖顺序；版本/哈希/签名校验；下载校验。
- P2：SQLite 历史（可选）；WinGet 集成；更多安装格式。

## 4. 技术栈与架构设计（阶段二暂定，非最终）

- Python 3.11+；PySide6（GUI）；subprocess（安装器/WinGet/PowerShell）；pywinauto（UIA）；PyAutoGUI（后备）；PyYAML；logging；sqlite3（可选）；PyInstaller（打包）。
- 模块边界草案：`core/`（清单模型、校验、下载、安装器执行、验证）、`automation/`（GUI 自动化）、`ui/`（PySide6，不含业务逻辑）、`tests/`。
- 关键设计：安装策略三层（本地优先 → 静默 → GUI 自动化 → 安全失败）；执行器在独立线程/进程，UI 不阻塞；每软件独立验证，不只看退出码。
- 说明：此为暂定，阶段二研究结束后再最终确定，不机械照搬。

## 5. 分阶段开发计划（可独立验收）

### 阶段一 研究与规划
- [x] 调研 WPKG 官方仓库与文档（FEASIBILITY.md §3）
- [x] 编写 FEASIBILITY.md 与 TODO.md
- 验收：分析引用真实文件路径；「无 GUI 自动化」结论有源码依据。

### 阶段二 架构定稿
- [ ] 冻结技术栈与依赖版本，输出依赖+许可证清单
- [ ] 设计 YAML 清单 schema（含校验）
- [ ] 设计模块边界与核心数据流
- 验收：技术选型附许可证核查；schema 可解析示例清单。

### 阶段三 核心功能（MVP）
- [ ] 实现 YAML 清单加载与 schema 校验
- [ ] 实现本地安装包检测（存在性/版本/哈希/签名）
- [ ] 实现下载器（URL 校验、下载、哈希/签名校验、失败不执行）
- [ ] 实现静默安装执行器（subprocess + 退出码白名单 + 超时 + 重启码处理）
- [ ] 实现安装结果独立验证
- [ ] 实现日志模块（分级、脱敏）
- 验收：用本地测试安装器跑通「本地存在→静默安装→验证成功」（测试场景 1）。

### 阶段四 GUI 自动化
- [ ] pywinauto 窗口/控件识别与操作框架
- [ ] 每软件独立 GUI 步骤配置 + 等待条件 + 超时 + 截图诊断
- [ ] PyAutoGUI 后备 + 安全停止策略
- [ ] 窗口正确性检测（避免误操作其他应用）
- 验收：测试窗口完成预设步骤；控件无法识别时安全停止（场景 8、9）。

### 阶段五 调度与 GUI 界面
- [ ] 依赖图与拓扑排序、冲突避免
- [ ] 批量调度、失败策略（继续/停止）、安全取消
- [ ] PySide6 界面（清单多选、状态、进度、日志、重试、统计）
- [ ] 耗时操作不阻塞主线程
- 验收：多选批量安装、进度/状态正确、取消安全（场景 10）。

### 阶段六 权限、安全与异常场景
- [ ] 受控提权流程（不绕过 UAC）
- [ ] 重启提示（不自动重启）
- [ ] 敏感信息脱敏、许可协议不自动接受
- 验收：管理员/重启场景正确提示（场景 11）。

### 阶段七 测试与打包
- [ ] 覆盖 12 类测试场景（用本地模拟安装器，不装真实第三方软件）
- [ ] PyInstaller 打包配置与 EXE 运行验证
- [ ] 中文 README（安装/配置/运行/测试/打包/已知限制）
- 验收：可重复测试流程 + EXE 在目标环境运行（场景 12）。

## 6. 许可证与第三方依赖审查

- 本项目 MIT；不复用 WPKG GPL 代码（只借鉴设计），若引用其 XML/代码片段须遵守 GPL-2.0。
- 拟用依赖及待核实的许可证（开发阶段逐一确认版本与条款）：PySide6 LGPL-3.0；pywinauto BSD-3-Clause；PyAutoGUI BSD-3-Clause；PyYAML MIT；PyInstaller GPL（打包豁免条款需确认）；sqlite3 Public Domain。
- 不宣称「无任何许可证义务」。

## 7. 已知风险、待验证事项及解决方案

| 风险/待验证 | 等级 | 解决方案 |
|---|---|---|
| GUI 自动化对自绘/Electron 界面失效 | 高 | 分级策略 + 安全失败 + 人工介入，不盲目点击 |
| 静默参数不通用 | 高 | 清单显式配置、示例标注，不假设统一参数 |
| 下载来源/校验安全 | 高 | URL 白名单+来源校验、下载后哈希/签名、失败不执行 |
| UAC/权限提升 | 中 | 受控提权，不绕过 |
| 版本比较边界（rc/beta） | 中 | 分段比较 + 显式版本策略 |
| PyInstaller GPL 豁免条款 | 中 | 开发期核实，必要时换打包方案 |
| 安装器需重启导致状态不一致 | 中 | 重启码白名单 + 独立验证 + 提示用户 |
| wpkg.org wiki 具体文档页（本次网络受限未抓取） | 低 | 开发期按需补齐，当前以仓库内 documents/ 与 xsd/ 为准 |
