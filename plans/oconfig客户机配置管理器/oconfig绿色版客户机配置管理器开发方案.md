# oconfig 绿色版客户机配置管理器开发方案

## 一、项目定位

本项目为**Odoo \+ Oharness \+ DSH 整套生态** 的**客户端绿色运维管理工具**。

替代原有笨重的 NW\.js 方案，改用 **Python \+ Tkinter 轻量GUI**，实现：

- 本地机器 Hosts 可视化管理、自动备份、校验、回滚

- Odoo\.conf 配置可视化修改、保存、校验

- Windows 服务启停、重启、状态查看（DSH/Oharness/Odoo 服务）

- 本地 DSH 客户端状态自检、连通性检测

- 对接 Oharness 服务端：任务查看、暂停任务审批、恢复任务

- 绿色单文件、免安装、Windows/Mac 双平台

**核心价值：客户机无需任何环境、无需安装、极小体积、纯运维管控，不承载浏览器内核，不重复占用 Chromium。**

---

## 二、整体架构关系（最终定型架构）

目前你整套系统的最终稳定架构如下（本次工具完全适配）：

1. **服务端 DSH**：多企业浏览器RPA池、Playwright、税局自动流程、状态机、多Session隔离

2. **客户端 DSH**：本地人机验证、本地浏览器调用、本地插件能力

3. **Oharness**：主控系统、任务编排、Odoo数据联动、权限、RPC调度、回调管理

4. **本工具（oconfig 客户机配置管理器）**：客户机本地GUI运维终端，**替代NW\.js**

架构流向：

```Plain Text
oconfig 客户机配置管理器(客户端GUI)
    ↓ HTTP/RPC
Oharness(主控) ←→ DSH服务端/客户端
    ↓
Odoo(业务数据/台账/待办)
```

---

## 三、技术选型（最终确定）

- 开发语言：Python 3\.10\+

- GUI框架：Tkinter（内置、零依赖、体积最小）

- 多页面架构：ttk\.Notebook 标签页分页

- Windows服务控制：pywin32（条件导入，不影响Mac打包）

- HTTP通信：requests（对接Oharness/DSH接口）

- 打包工具：PyInstaller

- 产物形态：**单文件绿色exe/可执行文件**

**体积控制：25–45MB，远优于NW\.js的200MB，无冗余Chromium**

### \.env 参数来源（本次新增）

所有路径参数统一从 `.env` 读取，关键变量：

| 变量 | 示例值 | 用途 |
|---|---|---|
| `O20_PATH` | `D:\odoo20-x64` | oconfig\.exe 编译产物部署目录；odoo\.conf / nginx\.conf 默认路径的推导根 |
| `O20_PY_PATH` | `D:\odoo20-x64\runtime\python3` | 开发期编译打包、运行期调用 Python 时使用的解释器目录（**目录**，解释器为其下 `python.exe`） |

\.env 查找顺序（开发期与运行时一致）：`exe(或源码)同级\.env` → `上级目录\.env` → `D:\oharness\.env`；全部找不到时回退按 exe 自身位置推导默认路径，保证绿色版拷到任意机器单文件可用。

解析方式：内置约 30 行轻量解析器（key=value、跳过注释与空行、去引号），**不引入 python\-dotenv**，避免污染 Odoo 共享运行环境。

界面中手动修改的路径写回 exe 同级 `oconfig_settings.json`，优先级高于 \.env 推导值。

---

## 四、软件页面结构（5大Page）

采用多标签页结构，功能完全隔离：

### Page0：配置管理（主页，本期实现）

绿色版最常改的 4 项配置集中在此页，均为"查看 \+ 快捷操作"，不做表单化编辑：

| 配置项 | 控件 | 行为 |
|---|---|---|
| odoo\.conf 文件 | 只读输入框（默认 `O20_PATH\odoo.conf`，即 exe 同级）\+ 浏览 \+ 「编辑」 | 编辑＝用记事本打开该文件；文件不存在时提示并置灰 |
| odoo 全局密码 | 显示当前 `admin_passwd`（默认打码，可切换明文）\+ 「重置」 | 重置＝弹窗输入新密码（确认两次）→ 时间戳备份 odoo\.conf → 原地改写 `admin_passwd` 行（缺行则在 `[options]` 段追加）→ 重载显示 |
| nginx\.conf 文件 | 只读输入框（默认 `O20_PATH\runtime\nginx\conf\nginx.conf`）\+ 浏览 \+ 「编辑」 | 同 odoo\.conf |
| hosts 文件 | 「以管理员打开 hosts」按钮 | `ShellExecuteW runas` 提权打开 `C:\Windows\System32\drivers\etc\hosts` 的记事本，本工具自身不提权 |

### Page1：本地进程管理器（核心，后续批次）

> 修正：绿色版各组件实际由 bat 脚本做进程编排（`r.bat`/`s.bat`/`db.bat` \+ `pv.exe`/`pg_ctl`/`tskill`），**并非 Windows 服务**。本页按进程模式实现，不再假设 sc/NSSM 服务。

- psutil 检测 python\(odoo\-bin\)、nginx、postgres 进程状态与端口占用

- 启停按钮直接调用对应 bat 脚本

- 状态自动刷新

Mac环境自动隐藏本页，不报错、不崩溃。

### Page2：Hosts 配置管理器

解决多Odoo站点、多本地域名切换、税局本地解析问题

- 自动读取系统 Hosts（Windows/Mac 自适应路径）

- 可视化列表展示所有解析

- 新增/删除/启用/禁用解析

- **自动备份（时间戳bak）**

- **修改后语法校验**

- **失败自动回滚**

### Page3：Odoo\.conf 配置管理器

可视化维护本地Odoo配置，避免手动改错导致服务无法启动

- 自动加载 odoo\.conf

- 常用配置项可视化：端口、数据库、日志、插件路径、admin密码、proxy模式

- 保存、重置、恢复默认

- 配置语法校验

- 修改后自动重启Odoo服务（可选）

### Page4：Harness 任务运维面板

对接 Oharness \+ DSH，实现客户端审批能力

- Oharness 服务地址配置

- 查看本机DSH任务列表

- 查看暂停任务（扫码/人脸验证待处理）

- 手动恢复任务、取消任务

- 本地DSH状态自检

### 全局底部：日志输出区

所有操作日志统一输出、滚动记录、方便排错。

---

## 五、核心安全机制（必须内置）

所有文件修改功能强制四件套：

1. **自动备份**（带时间戳）

2. **内容校验**

3. **写入失败自动回滚**

4. **权限检测**（管理员/root校验）

避免客户机改错 hosts、odoo\.conf 导致系统、Odoo、DSH 无法启动。

---

## 六、跨平台兼容方案

### Windows

- 启用 pywin32 服务管理

- hosts 路径：系统自动适配

- 管理员权限检测

### Mac

- 自动屏蔽Windows服务页功能

- hosts/odoo\.conf 功能完全可用

- Harness任务管理完全可用

- 可管理Mac的launchctl/brew服务（预留扩展）

---

## 七、打包方案（绿色版核心）

### Windows 打包命令

```Plain Text
pyinstaller --onefile --windowed --name oconfig main.py
```

输出：单文件 **oconfig\.exe**，免安装、绿色、无依赖。部署位置：`O20\_PATH` 根目录（即 `D:\odoo20-x64`）。

### Mac 打包命令

```Plain Text
pyinstaller --onefile --windowed --name oconfig main.py
```

### 打包特性

- \-\-windowed：无黑框GUI程序

- \-\-onefile：单文件分发，客户双击即用

- 纯净体积，无Chromium、无冗余运行库

---

## 八、与原有系统的融合优势（为什么彻底放弃NW\.js）

1. **不再双Chromium冲突**：客户机已有DSH\+Playwright浏览器，本工具零浏览器内核，不重复占用内存、磁盘

2. **体积从200MB降到30MB左右**

3. **打包快、分发快、更新快**

4. **专注运维，不抢DSH的浏览器执行职责**

5. **跨平台完美兼容**

6. **可长期迭代**：服务管理、配置管理、任务审批全部可扩展

---

## 九、开发排期（极简落地）

- **Day1**：多页面GUI框架搭建、日志系统、跨平台适配

- **Day2**：Windows服务管理模块开发

- **Day3**：Hosts配置备份/修改/校验/回滚模块

- **Day4**：Odoo\.conf可视化配置模块

- **Day5**：对接Oharness/DSH任务审批接口

- **Day6**：双平台打包、测试、BUG修复

---

## 十、最终系统闭环总结

现在你的整套架构是**极度干净、分层极致合理的最终版架构**：

- **DSH**：只做浏览器RPA、状态机、多会话、页面执行（纯执行层）

- **Oharness**：只做业务编排、任务调度、Odoo联动、权限、回调（纯业务主控）

- **oconfig 客户机配置管理器**：只做客户机运维、配置、服务启停、人工审批（纯本地运维终端）

三层完全解耦、各司其职、无冗余、无重复内核、无架构负担。

> （注：部分内容可能由 AI 生成）
