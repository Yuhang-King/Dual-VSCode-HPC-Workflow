# Dual-VSCode-HPC-Workflow

> 目标：新版本 VS Code 负责本地开发和 Codex，旧版本 VS Code 单独负责通过 Remote-SSH 连接老集群，做到本地与远端并行工作、互不干扰。

## 一页结论

| 项目 | 建议 |
| --- | --- |
| 新版本 VS Code | 只用于本地开发、Codex、普通插件。 |
| 旧版本 VS Code | 单独目录 + Portable Mode，只用于连接老集群。 |
| 为什么必须分开 | CentOS 7 / glibc 2.17 不满足 VS Code 1.99+ 的远端基线。 |
| 远端使用原则 | 在 VS Code 中编辑、轻量测试、看日志；正式重计算走集群调度。 |
| 空间原则 | `~/.vscode-server` 不要放在小容量 `home`，应迁到大盘再做软链接。 |
| Python / Jupyter 原则 | 先在终端确认 Conda 环境，再在 VS Code 中选择解释器；Notebook 只做轻量交互。 |

## 为什么要用双版本 VS Code

- Windows 的 VS Code Portable Mode 只支持 ZIP 包，不支持普通安装器。
- VS Code 1.99 开始，Remote Development 预编译 server 要求远端 Linux 满足更高基线。
- CentOS 7 常见环境是 `kernel 3.10`、`glibc 2.17`、`libstdc++ 3.4.19`，已经不满足新版要求。
- 所以更稳妥的做法是：
  - 新版 VS Code 留给本地开发。
  - 旧版 VS Code 1.98.x 专门连接老集群。

## 版本与下载建议

推荐策略：

- 本地开发：使用官方当前稳定版。
- 远端老集群：固定使用 `1.98.2` 的 Windows x64 ZIP。

说明：

- 本文档整理时，VS Code 更新页显示稳定版为 `1.115`，记录日期为 `2026-04-08`。

| 用途 | 推荐版本 | 官方链接 |
| --- | --- | --- |
| 本地开发 / Codex | 当前稳定版 | <https://code.visualstudio.com/download> |
| 当前版本说明 | 2026-04-08 记录为 `1.115` | <https://code.visualstudio.com/updates> |
| 旧版便携远程集群 | `1.98.2` Windows x64 ZIP | <https://update.code.visualstudio.com/1.98.2/win32-x64-archive/stable> |
| 备用旧版 | `1.97.2` Windows x64 ZIP | <https://update.code.visualstudio.com/1.97.2/win32-x64-archive/stable> |

## 推荐目录结构

```text
D:\
├─ VSCode-New\                           # 新版本，日常本地开发
└─ VSCode-CQJF\
   ├─ launch_cqjf_vscode.bat            # 旧版专用启动脚本（可选）
   └─ VSCode-win32-x64-1.98.2\
      ├─ Code.exe
      └─ data\                          # Portable Mode 关键目录
```

## 第 1 步：安装新版本 VS Code

1. 打开官方下载页：<https://code.visualstudio.com/download>
2. Windows 选择最新稳定版 `User Installer x64` 或 `System Installer x64`。
3. 正常安装即可。
4. 这个版本只用于本地开发、Codex、普通插件。
5. 不要用它去连接这台 CentOS 7 老集群。

## 第 2 步：安装旧版本 VS Code

1. 下载旧版 ZIP：<https://update.code.visualstudio.com/1.98.2/win32-x64-archive/stable>
2. 解压到：

```text
D:\VSCode-CQJF\VSCode-win32-x64-1.98.2
```

3. 确认以下文件存在：

```text
D:\VSCode-CQJF\VSCode-win32-x64-1.98.2\Code.exe
```

4. 在 `Code.exe` 同级创建空文件夹 `data`。
5. 只要 `data` 与 `Code.exe` 同级，VS Code 就会进入 Portable Mode。

## 第 3 步：旧版 settings.json

文件路径：

```text
D:\VSCode-CQJF\VSCode-win32-x64-1.98.2\data\user-data\User\settings.json
```

建议内容如下：

```json
{
  "update.mode": "none",
  "extensions.autoUpdate": false,
  "extensions.autoCheckUpdates": false,
  "remote.SSH.showLoginTerminal": true,
  "remote.SSH.useLocalServer": false,
  "remote.SSH.lockfilesInTmp": true,
  "window.restoreWindows": "none",
  "extensions.ignoreRecommendations": true,
  "update.showReleaseNotes": false,
  "remote.SSH.configFile": "C:\\Users\\wangyuhang\\.ssh\\config",
  "python.useEnvironmentsExtension": false
}
```

作用很简单：

- 锁定旧版，不让它自动升级。
- 让 Remote-SSH 更容易排查登录和兼容问题。
- 避免 Python 扩展的新环境管理逻辑干扰旧版远端场景。

## 第 4 步：旧版启动方式

如果你希望把旧版入口和新版彻底分开，可以放一个启动脚本：

```bat
@echo off
setlocal
set VSCODE_HOME=D:\VSCode-CQJF\VSCode-win32-x64-1.98.2
set VSCODE_EXE=%VSCODE_HOME%\Code.exe
if not exist "%VSCODE_EXE%" (
  echo [ERROR] Code.exe not found: %VSCODE_EXE%
  pause
  exit /b 1
)
start "" "%VSCODE_EXE%"
exit /b 0
```

备注：

- 这个脚本只是方便入口管理。
- 实际上直接双击 `Code.exe` 也可以启动旧版。

## 第 5 步：配置本地 SSH

SSH 配置文件：

```text
C:\Users\<your user name>\.ssh\config
```

示例：

```sshconfig
Host cqjf9
  HostName 10.144.65.9
  User <your user name>
  Port 22
```

先在 Windows 终端测试：

```powershell
ssh cqjf9
```

判断标准：

- 如果命令行能走到密码输入，再走到双因素验证码输入，说明 SSH 链路基本正常。
- 旧版 VS Code 的 Remote-SSH 主机列表就是从这份 `config` 读取的。

## 第 6 步：用旧版 VS Code 连接老集群

1. 打开旧版 VS Code。
2. 按 `Ctrl+Shift+X`，只安装必要扩展：
   - `Remote - SSH`
   - `Python`
   - `Jupyter`（只有确实需要 Notebook 时再装）
3. 按 `Ctrl+Shift+P`，执行 `Remote-SSH: Connect to Host...`
4. 选择 `cqjf9` 或你的实际主机名。
5. 第一次输入登录密码。
6. 第二次输入双因素验证码。
7. 如果看到 unsupported OS 警告，点击 `Allow`。
8. 连接成功后，左下角应该显示 `SSH: cqjf9`。

### 为什么新版会失败

你的远端集群如果是这类环境：

```text
CentOS 7 / kernel 3.10 / glibc 2.17 / libstdc++ 3.4.19
```

那么它不满足 VS Code `1.99+` 的远端最低基线。新版通常会直接报 prerequisites 不满足，因此旧版 `1.98.x` 才是兼容方案。

## 第 7 步：把 .vscode-server 迁到大盘

如果集群 `home` 空间小，这一步很重要。

原因：

- `.vscode-server` 很容易把 `home` 挤满。
- 最稳妥的做法是迁到项目盘或大容量目录，再在 `home` 下做软链接。

登录集群后执行：

```bash
pkill -f vscode || true
pkill -f "code-server" || true
pkill -f ".vscode-server" || true

rm -rf ~/.vscode-server ~/.vscode-remote
mkdir -p /<your project path>/.vscode-server
ln -sfn /<your project path>/.vscode-server ~/.vscode-server

ls -la ~/.vscode-server
readlink -f ~/.vscode-server
```

## 第 8 步：打开远端目录

- 连接成功后，直接使用 `File > Open Folder...` 打开远端项目目录。
- 优先打开大盘路径，不要把项目主要内容放在 `/home/`。
- 常见文件都可以直接编辑：`.py`、`.txt`、`.sh`、`.yaml`、`.md`、`.png`、`.jpg`、`.ipynb`。
- 如果命令面板里没有 `Remote-SSH: Open Folder...`，不一定是报错；只要左下角显示 `SSH: host`，直接用 `File > Open Folder...` 即可。

## 第 9 步：使用 Conda 环境

先在远端终端验证环境：

```bash
source ~/.bashrc
conda activate <conda_env_name>
python -V
which python
```

如果上面的命令正常，再回到 VS Code：

1. 执行 `Python: Select Interpreter`
2. 如果自动发现失败，选择 `Enter interpreter path`
3. 手动填写解释器路径，例如：

```text
/<your path to python>/bin/python
```

如果出现下面这类报错：

- `No environment managers registered`
- `Unable to handle .../python`

可以保留旧版配置中的这一项，然后重载窗口再试：

```json
"python.useEnvironmentsExtension": false
```

### 推荐：为工作区固定解释器

在项目目录创建 `.vscode/settings.json`：

```json
{
  "python.defaultInterpreterPath": "/<your path to python>/python"
}
```

## 第 10 步：使用 Jupyter / Notebook

先在远端环境中安装：

```bash
conda activate /<your path to conda env>/
conda install jupyter ipykernel
# 或者
pip install jupyter ipykernel
```

然后按下面的方法使用：

1. 在 VS Code 中直接打开远端 `.ipynb`
2. 点击右上角 Kernel 选择器
3. 选择远端 Conda 环境对应的解释器

使用原则：

- 轻量探索和交互分析可以直接在 Remote-SSH 窗口中运行。
- 正式训练、长时间任务、大规模计算不要在登录节点上跑。
- 重任务应申请交互节点或通过调度系统提交。

## 推荐工作流

| 场景 | 使用哪个 VS Code | 执行方式 |
| --- | --- | --- |
| 本地改代码 / Codex | 新版本 VS Code | 在本地项目目录工作，不连接老集群。 |
| 远端看代码 / 改脚本 / 查日志 | 旧版本 VS Code | Remote-SSH 连接集群，打开远端目录。 |
| 轻量 Python 测试 | 旧版本 VS Code + 远端终端 | `conda activate` 后做小样本运行。 |
| Notebook 交互分析 | 旧版本 VS Code + Jupyter | 打开 `.ipynb`，选择远端 Kernel。 |
| 正式训练 / 大计算 / 长任务 | 终端 + 调度系统 | 用 `qsub`、`sbatch`、`srun` 等方式提交；VS Code 只负责编辑和看结果。 |

## 常见问题与解决方法

### 1. 新版 VS Code 报 prerequisites 不满足

解决方式：改用旧版 `1.98.2` ZIP。

### 2. 卡在 Initializing VS Code Server

SSH 登录集群后清理旧的远端缓存，再重新连接：

```bash
rm -rf ~/.vscode-server ~/.vscode-remote
```

### 3. Remote-SSH 主机列表消失

检查两件事：

- `C:\Users\<your user name>\.ssh\config` 是否还在
- `settings.json` 中的 `remote.SSH.configFile` 是否仍然指向这份文件

### 4. 打开旧版后自动恢复上次远程窗口并报错

处理方式：

- 设置 `"window.restoreWindows": "none"`
- 删除 `workspaceStorage`
- 删除 `remote-ssh` 的 `globalStorage`

### 5. Python is not installed / No environment managers registered

处理方式：

1. 保留 `"python.useEnvironmentsExtension": false`
2. 执行 `Developer: Reload Window`
3. 手动填写解释器路径

### 6. home 被 .vscode-server 占满

处理方式：把 `~/.vscode-server` 迁移到大盘，再通过软链接连回 `home`。

## 常用修复命令

```bash
# 远端清理 VS Code Server
pkill -f vscode || true
pkill -f "code-server" || true
pkill -f ".vscode-server" || true
rm -rf ~/.vscode-server ~/.vscode-remote
ln -sfn /<your project path>/.vscode-server ~/.vscode-server

# 查看空间
du -sh ~
du -sh ~/.vscode-server
du -sh ~/.vscode-server/*

# Windows 侧杀残留
taskkill /F /IM Code.exe
taskkill /F /IM ssh.exe
```

## 最终执行清单

- 安装新版本 VS Code，只用于本地开发和 Codex。
- 下载旧版 `1.98.2` ZIP，解压到单独目录，并创建 `data` 文件夹。
- 配置旧版 `settings.json`，锁定更新和扩展行为。
- 配置 `C:\Users\wangyuhang\.ssh\config`。
- 用旧版 Remote-SSH 连接老集群并通过双因素认证。
- 把 `~/.vscode-server` 迁到大盘并建立软链接。
- 打开远端项目目录，尽量少装扩展。
- 在终端验证 `conda activate`、`python -V`、`which python`。
- 在 VS Code 中选择解释器，必要时手动指定路径。
- Notebook 只做轻量任务，正式任务走调度系统。

## 官方参考

- <https://code.visualstudio.com/docs/editor/portable>
- <https://code.visualstudio.com/download>
- <https://code.visualstudio.com/updates>
- <https://code.visualstudio.com/updates/v1_98>
- <https://code.visualstudio.com/docs/remote/ssh>
- <https://code.visualstudio.com/docs/remote/troubleshooting>
- <https://code.visualstudio.com/docs/remote/faq>
- <https://code.visualstudio.com/docs/remote/linux>
- <https://code.visualstudio.com/docs/python/environments>
- <https://code.visualstudio.com/docs/languages/python>
- <https://code.visualstudio.com/docs/datascience/jupyter-notebooks>
