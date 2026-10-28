# jam_ghost_deploy — 可行性分析

## 1. 项目概述

`jam_ghost_deploy` 是一个面向个人 Windows 桌面的无人值守批量软件安装器。用户通过 YAML 清单定义软件（静默安装参数、本地安装包路径、GUI 自动化步骤、验证规则），程序按「本地安装包优先 → 原生静默安装 → GUI 自动化 → 安全失败」的策略批量安装，并提供进度、日志、重试与结果验证。技术栈为 Python（GUI 拟用 PySide6）。

核心定位：借鉴 WPKG 的「清单驱动 + 安装状态检测」设计思想，但独立实现一个更适合个人桌面、带图形界面、具备 GUI 自动化回退能力的工具，**不复用 WPKG 源码**。

## 2. 需求分析

功能需求（自提示词提取）：

1. 本地安装包优先；本地不存在或不符合要求时才在线下载。
2. 优先使用软件原生静默安装参数。
3. 不支持静默安装时，用 GUI 自动化完成安装向导。
4. 可配置每个软件的安装路径、组件、快捷方式。
5. 批量安装、进度显示、日志、失败重试、安装结果验证。
6. Python 实现，结构清晰可维护可扩展。
7. 不复制 WPKG 源码或架构，独立实现。

约束：

- 优先 BSD/MIT/Apache-2.0 等宽松许可证依赖，核查所有依赖的许可证。
- README 中文；关键逻辑中文注释 + 类型标注。
- 不上传安装器/测试程序到未授权外部服务。
- 不绕过 UAC，不自动重启，不记录密码/令牌等敏感信息。

验收标准（最终交付）：完整 Python 源码、真实进度的 `TODO.md`、`README.md`、示例软件清单及规则说明、依赖+许可证说明、自动化测试、Windows EXE 打包配置、安装日志与失败诊断能力。

## 3. 现有方案调研（WPKG 深度分析）

[verified] 官方仓库为 `github.com/wpkg/wpkg-js`（描述「automated software deployment for Windows workstations, driven by XML package, profile and host definitions」，默认分支 `master`，最近推送 2026-09-08，许可证 **GPL-2.0**）。提示词给出的 `github.com/sonicnkt/wpkg` 返回 **404**，已按预案改用官方仓库。

### 3.1 架构与文件结构

WPKG 是客户端-服务器模型的 XML 驱动部署工具。核心文件（均从仓库实际读取）：

- `wpkg.js` — 主引擎（JScript，经 cscript/wscript 运行，见 config.xml 注释；仓库含 Node 测试套件 `tests/*.test.js`，覆盖 `checkCondition`、`versionCompare`、`parseISODate`、`getIPAddresses`、`getMACAddresses`、`packageNodes`、`checkHostAttribute`、`filterConditionalNodes` 等函数）。
- `config.xml` — 全局设置（`<param name= value=>`）。
- `packages.xml` + `packages/` — 软件包定义。
- `profiles.xml` + `profiles/` — 配置集（软件包分组）。
- `hosts.xml` + `hosts/` — 主机→配置集映射。
- `xsd/*.xsd` — config/packages/profiles/hosts/settings 的 XML Schema。
- `tools/` — downloader.cmd、execute-elevated.js（提权）、64-bit-wrapper、install-status-report。
- `documents/`、`site/xsd-doc/` — 文档。

### 3.2 十项分析（源码依据）

1. **软件包定义与清单管理**：`<package id= name= revision= reboot= priority= execute= notify=>`，`id` 唯一、供 profiles 引用（`packages.xml` 头部注释）；支持 `<include package-id=>` 复用（`packages/Templates/template_MSI.xml`）。

2. **EXE/MSI/脚本安装方式**：`<command type="install|upgrade|remove|downgrade" cmd= include=>`。MSI：`msiexec /i /qn`（`packages.xml`）；模板 `MsiExec.exe /passive /norestart /log`（`template_MSI.xml`）。脚本：`cmd.exe /c`、`net time`、`pnputil.exe /add-driver`（`packages.xml`、`examples/packages/execute-check-with-download.xml`）。

3. **静默参数组织与执行**：静默参数直接写在 `cmd` 属性，配合变量。InnoSetup 模板给出 `/SP- /VERYSILENT /SUPPRESSMSGBOXES /LOG /NORESTART`（`template_InnoSetup.xml`）；通用 EXE `/quiet /install`（`packages.xml`）。结论：WPKG 只按字符串执行命令，静默参数完全由清单作者保证正确，工具不校验参数有效性。

4. **依赖与安装顺序**：`<depends package-id=>` 强制前置安装（`packages.xml`）；`priority` 属性控制执行先后，但注释明确「推荐用 depends 而非 priority 保证顺序」（`packages.xml`）。

5. **已安装状态/版本/重复安装**：`<check>` 支持 `type=registry|file|uninstall|execute|logical`、`condition=exists|versiongreaterorequal|exitcodeequalto|not`（`packages.xml`、`msi-with-exit-codes.xml`、`conditional-commands.xml`、`execute-check-with-download.xml`）。默认卸载注册表路径 `SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall`（config.xml `sRegPath`）。版本比较用 `versionCompare`（`tests/versionCompare.test.js`），支持 `volatileReleaseMarker`（rc/beta 等，config.xml）。无 check 的包只能靠退出码推断（`packages.xml` wpkg2 注释）。

6. **失败/退出码/日志/重试**：`<exit code= reboot=>` 声明允许的退出码；`0` 成功（隐式），`3010`=已装需重启（reboot=postponed）、`1641`=安装器触发重启、`1605`=产品未安装、`259`=无更多项、`any`（`msi-with-exit-codes.xml`、`execute-check-with-download.xml`、`packages.xml`）。命令超时默认 3600s（config.xml `execTimeout`）。日志：`logLevel` 位掩码（1 错误/2 警告/4 信息/8 审计成功/16 审计失败）、`log_file_path` 默认 `%TEMP%`、`logfilePattern` 默认 `wpkg-[HOSTNAME].log`（config.xml）。重试是「下次运行自动重试」（check 失败则重装），无同次内重试（`packages.xml` 注释）。

7. **本地/远程软件包**：`<download url= target= timeout=>`；包级 download 在第一条命令前执行，check 级 download 用于拉取检测脚本（`execute-check-with-download.xml`）。`%SOFTWARE%` 通常指向网络共享；`wpkg_base` 可指向 HTTP；`noDownload` 可禁用下载（config.xml）。设计以域内 SMB 共享为中心，非面向公网下载。

8. **配置格式与扩展**：XML + `<param>`、`<variables><variable name= value= architecture= os= lcid=>`、`<languages>`、`packages_path|profiles_path|hosts_path` 支持多路径（管道分隔，文件/目录/HTTP 混合，config.xml）。有 XSD 校验（`xsd/`）。

9. **WPKG 的不足（不适合个人桌面的设计）** [inferred]：
   - GPL-2.0 传染性许可证；若复用源码需整体 GPL（本项目只借鉴设计、不复制代码，规避）。
   - 无图形界面，只有 CLI/服务 + 可选 `sendStatus` STDOUT 供外部 GUI 读取（config.xml），无多选清单、无进度条 UI。
   - 以域/网络共享为中心（hosts/profiles、SMB 路径），对单机本地文件场景冗余。
   - 无 GUI 自动化回退（见第 10 点），强依赖安装器支持静默参数。
   - XML 冗长、学习成本高；版本比较为字符串比较，存在 rc/beta 边界情况（config.xml `volatileReleaseMarker` 注释自述）。

10. **GUI 自动化能力** [verified]：**WPKG 不支持**。整个仓库树无任何窗口/控件自动化代码；`tools/` 仅 downloader、execute-elevated、64-bit-wrapper、install-status-report。config.xml 的 `quiet`/`nonotify` 只是抑制控制台/弹窗输出，与向导点击无关。因此「脚本执行 ≠ GUI 自动化」：WPKG 只能执行静默安装器，遇到不支持静默的向导型安装器（自绘界面、UAC、许可协议、需重启）无法自动点击，只能失败。这正是本项目需独立实现的能力。

## 4. 技术可行性

- 业务逻辑 Python：成熟可行。
- GUI PySide6：Qt 官方绑定，桌面 GUI 成熟。
- subprocess 启动安装器/msiexec/WinGet：Windows 原生，可行。
- pywinauto（UIA/Win32 后端）做窗口/控件自动化：可行，但对自绘/Electron 界面受限。
- PyAutoGUI（屏幕坐标/图像）作后备：可行但脆弱，需严格控制。
- PyYAML 清单、logging 日志、sqlite3 持久化（可选）、PyInstaller 打包：均可行。

结论：技术可行，核心难点在 GUI 自动化的鲁棒性（窗口识别、等待条件、错误恢复）与不同安装器的静默参数适配，非语言/框架层面。

## 5. 许可证

- 本项目：**MIT**（默认最宽松常用许可证）。因不复制 WPKG GPL 代码、仅借鉴设计思想，无许可证传染问题。**注意**：若未来直接搬运 WPKG 的 XML 模板/代码片段，须遵守 GPL-2.0。
- 拟用依赖许可证（初步判断，开发阶段逐一核实版本与确切条款）：
  - PySide6 — LGPL-3.0（Qt 官方绑定）。
  - pywinauto — BSD-3-Clause。
  - PyAutoGUI — BSD-3-Clause。
  - PyYAML — MIT。
  - PyInstaller — GPL（含打包豁免条款，需核实具体版本）。
  - sqlite3（Python 标准库）— Public Domain。
  不宣称任何软件「无任何许可证义务」。

## 6. 风险

| 风险 | 等级 | 缓解 |
|---|---|---|
| GUI 自动化脆弱（自绘界面/UAC/安全确认/重启） | 高 | 分级策略 + 安全失败 + 人工介入，不盲目点击 |
| 静默参数不通用（各安装器不同） | 高 | 清单显式配置、示例标注，不假设统一参数 |
| 下载来源与校验安全 | 高 | URL 来源校验、下载后哈希/签名校验、失败不执行 |
| 权限提升与 UAC | 中 | 受控提权，不绕过 |
| 版本比较边界 | 中 | 分段比较 + 显式版本策略 |
| 许可证合规 | 中 | 开发期持续核查依赖 |

## 7. 推荐方案

借鉴 WPKG 的：清单驱动、`<check>` 式安装状态检测、install/upgrade/remove 命令分离、退出码白名单、depends 依赖、变量+条件。独立实现为：YAML 清单、三优先级安装策略（本地优先/静默/GUI 自动化）、PySide6 图形界面、可选 SQLite 历史、PyInstaller 打包。**不复用 WPKG 任何源码**。

## 8. 结论

**GO**。技术可行、范围清晰，核心风险（GUI 自动化鲁棒性、安装器适配）可通过分级策略与「安全失败 + 人工介入」缓解。按 TODO.md 分阶段推进。

## 9. 参考资料

- [verified] https://github.com/wpkg/wpkg-js （官方仓库；提示词给的 sonicnkt/wpkg 已 404）
- [verified] 仓库内文件（均实际读取）：`config.xml`、`packages.xml`、`examples/packages/{msi-with-exit-codes,execute-check-with-download,conditional-commands}.xml`、`packages/Templates/{template_MSI,template_InnoSetup}.xml`、`xsd/`、`tests/*.test.js`
- [to confirm] https://wpkg.org/ （官方站点；`packages.xml` 头部引用其 `Category:Silent_Installers` 与 `Category:Documentation`；本次网络受限未直接抓取 wiki 页面，具体页面内容待开发期补齐）

## 假设

- 项目名采用提示词暂定名 `jam_ghost_deploy`（未调整）。
- 目标平台为个人 Windows 10/11 x64 桌面。
- 仓库公开（默认）；不上传任何安装器/测试程序。
