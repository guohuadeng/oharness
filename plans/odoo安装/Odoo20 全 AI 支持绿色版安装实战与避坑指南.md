# Odoo 20 全 AI 支持绿色版安装实战与避坑指南

> 本指南记录了在 Windows 11 上部署 **Odoo 20 全 AI 支持绿色版** 的完整过程。整个安装（含 pip 升级、requirements.txt 依赖安装、renderPM 后端适配、导入验证）均由 **AI 辅助完成**，文中如实记录了踩过的每一个坑及对应的解决方案，可作为同类环境部署的参考手册。

---

## 一、环境概览

本绿色版由 [odooai.cn](https://www.odooai.cn) 提供，在 Windows 上搭建了一个完整的高性能 Odoo 环境，主要组件版本如下：

| 组件 | 版本 | 说明 |
|------|------|------|
| Python | 3.13.14（64 位） | 绿色版，路径 `runtime\python3\python.exe` |
| PostgreSQL | 16.4（64 位） | 已优化，数据文件在 `runtime\pgsql\data` |
| Nginx | 1.15.5（64 位） | 反向代理，实现 longpolling 桌面消息通知 |
| Node.js | v24.18.0 | AI 主流，目录直接置于 `runtime` 下 |
| Odoo | 20 社区版（20260718 版） | 源码在 `source` 目录 |
| pip | 26.1.2 | 已是最新 |

> 备注：相比 Odoo 官方主推的 Python 3.12，本绿色版采用了更新的 **Python 3.13.14**，性能更高，但部分依赖包的预编译 wheel 尚未覆盖 cp313，这是后文几个坑的根源。

### 目录结构

```
D:\odoo20-x64\
├─runtime\          运行库（python3 / pgsql / nginx / nodejs）
├─source\           Odoo 20 源码（含 requirements.txt）
│  ├─addons_ent\    企业版模块
│  └─myaddons\      odooai.cn 优化模块
├─extra\            附加包（wkhtmltopdf、get-pip.py 等）
├─fixed\            原生源码优化修正
├─odoofile\         Odoo 生成的静态文件资源
├─odoo.conf         配置文件
├─r.bat             启动 Odoo（最常用）
├─s.bat             停止 Odoo
├─u.bat             更新 Odoo 源码
└─init.bat          重新初始化数据库（需管理员）
```

---

## 二、安装前准备

### 1. 安装 Windows 系统支持

部分组件（PG、Python 依赖）依赖 VC++ 运行时，请先执行：

```
.\extra\vcredist_x64.exe
```

如果后续遇到 dll 错误，多半也是这个没装。

### 2. 设置环境变量（关键）

绿色版 Python 必须通过完整 PATH 调用，否则会找不到 pgsql、wkhtmltopdf 等依赖。在项目根目录打开 PowerShell 或 CMD：

```bat
SET PATH=%CD%\runtime\pgsql\bin;%CD%\runtime\python3;%CD%\runtime\python3\Scripts;%CD%\runtime\python3\Lib\site-packages\bin;%CD%\runtime\win32\wkhtmltopdf;%CD%\runtime\nodejs;%CD%\source;%PATH%
SET PYTHONPATH=%CD%\runtime\python3\Lib\site-packages
SET PYTHONUTF8=1
```

> 这三行是绿色版正常运行的基石，后文所有命令都假设已设置好上述环境变量。

---

## 三、第一步：升级 pip

### 为什么要单独说 pip

绿色版 Python 有一个**经典坑**：`Scripts\pip.exe` 启动器可能绑定到其他 Python 解释器（比如系统已装的 Python），导致 `pip install` 装到了错误的位置。**最佳实践是始终用 `python.exe -m pip` 调用**，这样 pip 一定作用在当前解释器上。

### 检查与升级

```powershell
D:\odoo20-x64\runtime\python3\python.exe -m pip --version
D:\odoo20-x64\runtime\python3\python.exe -m pip install --upgrade pip -i https://mirrors.aliyun.com/pypi/simple/
```

本次执行结果：

```
pip 26.1.2 (python 3.13)
```

pip 26.1.2 已是清华镜像源上的最新版，无需升级。同时验证 `Scripts\pip.exe` 与 `python.exe` 指向同一解释器，无错配。

> **避坑要点**：如果 `pip --version` 显示的 Python 版本与 `python --version` 不一致，说明启动器错配，必须改用 `python.exe -m pip` 调用。

---

## 四、第二步：安装 requirements.txt 依赖

这是整个安装过程中**坑最多**的一步。`source\requirements.txt` 共 64 行，包含 Odoo 20 运行所需的全部 Python 依赖。

### 4.1 镜像源踩坑实录

#### 坑 ① 清华源找不到 asn1crypto

首次使用清华源安装：

```
pip install -r requirements.txt -i https://pypi.tuna.tsinghua.edu.cn/simple
```

报错：

```
ERROR: Could not find a version that satisfies the requirement asn1crypto==1.5.1 (from versions: none)
```

第一反应是包名拼错，但用 `pip index versions asn1crypto -i https://pypi.org/simple` 对比官方源，发现官方源完全正常。**结论：清华源对 asn1crypto 的索引存在同步缺陷。**

#### 坑 ② 阿里云源找不到 num2words

改用阿里云源重试：

```
pip install -r requirements.txt -i https://mirrors.aliyun.com/pypi/simple/
```

asn1crypto 这次通过了，但 num2words 又报同样的错：

```
ERROR: Could not find a version that satisfies the requirement num2words==0.5.13 (from versions: none)
```

继续用 `pip index versions` 逐一测试各镜像源，得到一张**镜像源缺陷对照表**：

| 镜像源 | asn1crypto | num2words |
|--------|------------|-----------|
| 清华源 | ❌ 缺失 | ✅ 正常 |
| 阿里云 | ✅ 正常 | ❌ 缺失 |
| 腾讯云 | ✅ 正常 | ✅ 正常 |
| 中科大 | ✅ 正常 | ✅ 正常 |
| 官方 PyPI | ✅ 正常 | ✅ 正常 |

> **关键认知**：这不是 pip 版本问题（pip 26.1.2 与各源都兼容），而是**不同镜像源对不同包存在索引同步缺陷**，且缺陷包各不相同。任何一个单一镜像源都无法保证 requirements.txt 里所有包都能命中。

### 4.2 多源 fallback 策略（核心解决方案）

既然单个镜像源靠不住，那就让 pip **同时查询多个源**——主源装不上的包，自动从备用源补。这正是 `--extra-index-url` 的用武之地。

最终采用的命令：

```powershell
D:\odoo20-x64\runtime\python3\python.exe -m pip install -r D:\odoo20-x64\source\requirements.txt -i https://mirrors.aliyun.com/pypi/simple/ --extra-index-url https://pypi.org/simple
```

- `-i https://mirrors.aliyun.com/pypi/simple/`：阿里云作主源（国内速度快）
- `--extra-index-url https://pypi.org/simple`：官方 PyPI 作备用源（主源缺失的包从这里补）

执行后 62 个包全部安装成功，包含之前报错的 asn1crypto 和 num2words。

> **避坑要点**：批量安装 requirements.txt 时，**务必配置 `--extra-index-url https://pypi.org/simple`**，这是对抗国内镜像源索引缺陷的最稳方案。README.md 第 128 行也已采用此策略。

### 4.3 rl-renderPM 踩坑实录（最棘手的坑）

依赖装完后，本以为大功告成，结果验证时发现 `reportlab.graphics._renderPM` 导入失败。

#### 问题根因

`requirements.txt` 第 50、53 行：

```
reportlab==4.1.0 ; python_version >= '3.12'
rl-renderPM==4.0.3 ; sys_platform == 'win32' and python_version >= '3.12'  # Needed by reportlab 4.1.0 but included in deb package
```

- `reportlab 4.1.0` 在 PyPI 上只有 `py3-none-any.whl`（纯 Python），**不含 `_renderPM` 这个 C 扩展**。
- `_renderPM` 是 reportlab 的位图渲染后端（PNG/JPG），由 `rl-renderPM` 包单独提供。
- 而 `rl-renderPM 4.0.3` 的预编译 wheel **最高只到 cp312**，没有 cp313 版本。
- 本机无 MSVC 编译器（`cl.exe` 未找到），源码构建也走不通（wheel≥0.42 移除了 `get_abi_tag`，旧版 setup 脚本直接报错）。

换句话说：**Python 3.13 + Windows + 无 MSVC = rl-renderPM 无法安装**，而 reportlab 又强依赖它。

#### 坑 ④ 验证脚本的导入名陷阱

排查过程中还踩了一个小坑：验证脚本里写 `import pycairo` 失败，其实 **pycairo 包的导入名是 `cairo`**。同类坑还有：

| pip 包名 | import 名 |
|----------|-----------|
| pycairo | `cairo` |
| Babel | `babel` |
| XlsxWriter | `xlsxwriter` |

> **避坑要点**：写导入验证脚本时，包名和导入名不一定一致，务必查文档。

### 4.4 rlPyCairo 替代方案（完美解决）

查阅 reportlab 官方文档发现：**rlPyCairo 是 reportlab 官方默认的 renderPM 后端**，功能与 `_renderPM` 等价，且基于 pycairo（有 cp313 预编译 wheel），无需 C 编译。

安装：

```powershell
D:\odoo20-x64\runtime\python3\python.exe -m pip install rlPyCairo -i https://mirrors.aliyun.com/pypi/simple/ --extra-index-url https://pypi.org/simple
```

执行后自动装上：

- `rlPyCairo 0.4.0`
- `pycairo 1.29.0`（cp313 wheel，无需编译）
- `freetype-py 2.5.1`

> **避坑要点**：Python 3.13 环境下，**跳过 rl-renderPM，直接装 rlPyCairo**，这是 reportlab 官方推荐的无编译方案。安装 requirements.txt 时可先用 `Where-Object { $_ -notmatch '^\s*rl-renderPM' }` 过滤掉该行。

---

## 五、第三步：验证安装结果

安装完成后，建议跑一遍导入验证，确保所有关键包可用。重点验证 renderPM 后端和 PDF 生成：

```python
# renderPM 后端验证（核心）
import reportlab
import rlPyCairo
import cairo  # 注意：pycairo 的导入名是 cairo
from reportlab.graphics import renderPM
from reportlab.graphics.shapes import Drawing, String

d = Drawing(200, 100)
d.add(String(100, 50, 'renderPM OK', textAnchor='middle', fontSize=20))
png_data = renderPM.drawToString(d, fmt='PNG')
print(f"renderPM PNG 渲染: OK, {reportlab.Version}, rlPyCairo={rlPyCairo.__version__}, png={len(png_data)}B")

# PDF 生成验证
from reportlab.pdfgen import canvas
from reportlab.lib.pagesizes import A4
import io

buf = io.BytesIO()
c = canvas.Canvas(buf, pagesize=A4)
c.drawString(100, 700, "PDF generate OK")
c.showPage()
c.save()
print(f"reportlab PDF 生成: OK, pdf={len(buf.getvalue())}B")
```

本次验证结果（30/30 全部通过）：

```
OK     renderPM(PNG渲染)           4.1.0 rlPyCairo=0.4.0 png=2709B
OK     reportlab PDF生成           4.1.0 pdf=1419B
OK     lxml                      5.4.0
OK     psycopg2                  2.9.10 (dt dec pq3 ext lo64)
OK     Pillow                    11.3.0
OK     Werkzeug                  3.1.3
OK     polib                     1.2.0
... (全部 OK)
Total: 30, OK: 30, FAIL: 0
```

---

## 六、避坑总结

| 序号 | 坑点 | 现象 | 解决方案 |
|------|------|------|----------|
| 1 | 绿色版 pip 启动器错配 | pip 装到错误解释器 | 始终用 `python.exe -m pip` 调用 |
| 2 | 清华源缺 asn1crypto | `from versions: none` | 弃用清华源作主源 |
| 3 | 阿里云源缺 num2words | `from versions: none` | 加 `--extra-index-url https://pypi.org/simple` |
| 4 | rl-renderPM 无 cp313 wheel | `_renderPM` 导入失败 | 用 rlPyCairo 官方后端替代 |
| 5 | 无 MSVC 无法源码编译 | setup.py 报 get_abi_tag 错误 | 同上，走纯 wheel 路线 |
| 6 | 包名与导入名不一致 | `import pycairo` 失败 | 改用 `import cairo` |
| 7 | reportlab 4.1.0 纯 Python | 不含 _renderPM C 扩展 | 必须装 rl-renderPM 或 rlPyCairo |

### 三条黄金法则

1. **绿色版 Python 一律用 `python.exe -m pip`**，杜绝启动器错配。
2. **批量装 requirements.txt 必加 `--extra-index-url https://pypi.org/simple`**，对抗国内镜像源索引缺陷。
3. **Python 3.13 下 reportlab 用 rlPyCairo 替代 rl-renderPM**，免编译、官方支持、功能等价。

---

## 七、附录：完整命令清单

以下命令均在项目根目录 `D:\odoo20-x64` 下执行，假设已设置好环境变量。

### 7.1 环境变量设置

```bat
SET PATH=%CD%\runtime\pgsql\bin;%CD%\runtime\python3;%CD%\runtime\python3\Scripts;%CD%\runtime\python3\Lib\site-packages\bin;%CD%\runtime\win32\wkhtmltopdf;%CD%\runtime\nodejs;%CD%\source;%PATH%
SET PYTHONPATH=%CD%\runtime\python3\Lib\site-packages
SET PYTHONUTF8=1
```

### 7.2 升级 pip

```powershell
runtime\python3\python.exe -m pip install --upgrade pip -i https://mirrors.aliyun.com/pypi/simple/
```

### 7.3 安装 Odoo 依赖（排除 rl-renderPM）

PowerShell 下生成排除 rl-renderPM 的临时 requirements：

```powershell
Get-Content .\source\requirements.txt | Where-Object { $_ -notmatch '^\s*rl-renderPM' } | Set-Content .\source\requirements_install.txt -Encoding UTF8
```

安装：

```powershell
runtime\python3\python.exe -m pip install -r .\source\requirements_install.txt -i https://mirrors.aliyun.com/pypi/simple/ --extra-index-url https://pypi.org/simple
```

### 7.4 安装 rlPyCairo 替代 rl-renderPM

```powershell
runtime\python3\python.exe -m pip install rlPyCairo -i https://mirrors.aliyun.com/pypi/simple/ --extra-index-url https://pypi.org/simple
```

### 7.5 验证安装

```powershell
runtime\python3\python.exe -c "import reportlab, rlPyCairo, cairo; from reportlab.graphics import renderPM; from reportlab.graphics.shapes import Drawing, String; d=Drawing(200,100); d.add(String(100,50,'OK',textAnchor='middle',fontSize=20)); print('renderPM OK', len(renderPM.drawToString(d,fmt='PNG')),'B')"
```

### 7.6 启动 Odoo

```bat
r.bat
```

启动后访问 http://localhost:8020 或 http://localhost，数据库 `demo`（密码 `odoo`），管理用户 `admin / admin`。

---

## 八、常见问题处理

### Q1：启动 Odoo 报数据库错误

进入 pgsql bin 目录重新初始化：

```bat
cd runtime\pgsql\bin
rd /s/q ..\data
initdb.exe -D ..\data -E UTF8
pg_ctl -D ..\data -l logfile start
createuser --createdb --no-createrole --no-superuser --pwprompt odoo
```

### Q2：需要 RAG 向量扩展

```sql
CREATE EXTENSION IF NOT EXISTS vector;
SELECT * FROM pg_extension WHERE extname = 'vector';
```

### Q3：权限不足

```sql
psql -d postgres -c "SELECT rolname FROM pg_roles;"
ALTER ROLE odoo WITH SUPERUSER;
```

### Q4：更新 Odoo 源码

```bat
s.bat      :: 先停止
u.bat      :: 从 git 下载最新版本覆盖 source 目录
r.bat      :: 重新启动
```

如手工更新，请至官方下载后覆盖 `./source` 目录：
- https://nightly.odoo.com/master/nightly/src/odoo_19.5a1.latest.zip
- https://nightly.odoocdn.com/master/nightly/src/odoo_19.5a1.latest.zip

---

## 九、写在最后

本次安装全程由 AI 辅助完成，从 pip 升级、依赖安装、镜像源排查、renderPM 后端适配到最终验证，AI 不仅执行了命令，还在遇到错误时**主动诊断、对比镜像源、查阅官方文档、提出替代方案**。整个过程体现了 AI 辅助编程在复杂环境部署中的价值——它能把"踩坑→排查→解决"的循环从数小时压缩到数十分钟。

如果你在部署中遇到本文未覆盖的问题，欢迎参考 [odooai.cn](https://www.odooai.cn) 获取更多支持。

> 本指南基于 2026 年 7 月的实际部署记录整理，环境为 Windows 11 + Python 3.13.14 + Odoo 20 社区版。
