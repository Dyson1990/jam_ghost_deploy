# jam_ghost_deploy

Windows 无人值守批量软件安装器（Python）。

> 当前状态：项目初始化阶段，已完成可行性分析与开发计划（见 `FEASIBILITY.md`、`TODO.md`），业务代码尚未实现。

## 目标

按「本地安装包优先 → 原生静默安装 → GUI 自动化 → 安全失败」的策略批量安装软件，提供进度、日志、重试与结果验证。

## 计划功能

- YAML 软件清单（静默参数、本地路径、GUI 步骤、验证规则）。
- 批量安装、进度显示、日志、失败重试、结果验证。
- 本地安装包优先 + 下载校验。
- GUI 自动化（pywinauto 为主，PyAutoGUI 后备）。
- PySide6 桌面界面。

## 当前进度

- [x] WPKG 调研（`FEASIBILITY.md` §3）
- [x] 可行性分析与开发计划（`FEASIBILITY.md`、`TODO.md`）
- [ ] 业务代码（见 `TODO.md` 阶段二～七）

## 技术栈（暂定）

Python 3.11+、PySide6、pywinauto、PyAutoGUI、PyYAML、logging、sqlite3（可选）、PyInstaller。

## 安装 / 配置 / 运行 / 测试 / 打包

待业务代码落地后补充（见 `TODO.md` 阶段七）。

## 已知限制

- 通用 GUI 自动化对自绘界面、UAC、许可协议、需重启的安装器，可能需要专门适配或人工介入（不夸大兼容性）。
- 静默参数因安装器而异，需在清单中显式配置，不假设统一参数。

## 许可证

[MIT](LICENSE)。不包含 WPKG 的 GPL 代码。
