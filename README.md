# Open Record

按 Topic 整理对话、操作步骤、结果与证据边界。每个 Topic 独立成块，通过下方索引跳转。

<a id="index"></a>

## Index · 主题索引

| Topic | 内容 | 记录日期 |
|---|---|---|
| [01 · iMac 双 OpenClaw](#topic-01) | 双实例配置、统一脚本、手工启动与验证边界 | 2026-08-31 / 2026-09-28 |
| [02 · ChIA-DropBox2 命令参考](#topic-02) | 仓库已有的 cdb2 参数、示例与输出说明 | 未注明 |

---

<a id="topic-01"></a>

## Topic 01 · iMac 双 OpenClaw

### 项目范围

本页只记录 iMac 上同时运行两个 OpenClaw 实例的部署结果、启动方式和验证边界。

### 2026-08-31：两套实例的启动方式

> 用户：btw：如果我要是重启imac机器，那我用什么命令分别重启这两个openclaw呢？

操作与决定：将迁移实例放在独立服务账户 `openclaw2` 下，使用独立 Gateway，与原 iMac 实例共同使用 Ollama。

迁移完成后，iMac 上保留两套独立 Gateway：

| 实例 | 服务标识 | 端口 |
|---|---|---:|
| 原 iMac OpenClaw | `system/ai.openclaw.imac-existing` | `18789` |
| 从 M1 迁移的第二实例 | `system/ai.openclaw.migrated-m1` | `19789` |
| 共享 Ollama | 两个实例的模型服务 | `11434` |

结论与边界：历史记录确认两套实例可同时运行；本次整理没有重新连接 iMac 检查当前运行状态。

### 2026-08-31：改为重启后手动启动

> 用户：不用让系统重启自动加载；你给我做个简单的命令script；每次重启后我手工运行加载两个openclaw

操作与决定：安装统一启动脚本，并将两套服务的 `RunAtLoad` 和 `KeepAlive` 均设为 `false`。

两套服务均配置为不随系统重启自动启动。手工启动命令为：

```bash
start-two-openclaw
```

完整路径为：

```bash
/usr/local/bin/start-two-openclaw
```

脚本会检查并启动共享 Ollama，加载两套 LaunchDaemon，分别启动 `18789` 和 `19789`，最后检查三个端口是否监听成功。

#### 当日验证结果

> 助手：手动脚本测试成功：卸载后，仅运行 `start-two-openclaw` 就恢复了共享 Ollama、`18789` 和 `19789`。

- 两个 Gateway 均可同时监听：`18789`、`19789`。
- 两个 QQBot 在最终验证中均为 `running=true`、`connected=true`、`lastError=null`。
- 两个实例共享本机 Ollama；迁移实例使用 `ollama/qwen3.5:cloud`。
- 源机器上的旧 OpenClaw 已停止，避免同一个 QQBot 被两台机器重复连接。

证据边界：以上为 2026-08-31 的历史验证结果，并非当前健康检查。统一脚本会对两套服务执行 `kickstart -k`，因此也会重启已经运行的原实例。

### 2026-09-28：查找第二实例命令与 GitHub 记录

> 用户：iMac 上完成过“双 OpenClaw”方案：用什么命令启动第二个openclaw？这部分有没有上传到github

操作：回查原始迁移会话，确认统一脚本及第二实例的服务名称。

**命令纠正：`start-two-openclaw` 会启动或重启两个实例，不是只启动第二个。**

如果第二实例的服务已加载，只启动它（不强制重启已有进程）：

```bash
sudo launchctl kickstart system/ai.openclaw.migrated-m1
```

需要重启第二实例时：

```bash
sudo launchctl kickstart -k system/ai.openclaw.migrated-m1
```

如果提示找不到服务，先加载再启动：

```bash
sudo launchctl bootstrap system /Library/LaunchDaemons/ai.openclaw.migrated-m1.plist
sudo launchctl kickstart system/ai.openclaw.migrated-m1
```

上述单实例命令依据当时安装的服务配置整理，本轮未在 iMac 上重新执行。单独启动前需确保共享 Ollama 已运行；可在 iMac 执行以下只读检查：

```bash
sudo lsof -nP -iTCP:11434 -iTCP:19789 -sTCP:LISTEN
```

结论与边界：历史记录未提供该项目的 GitHub 提交或推送证据。此前尝试读取目标仓库时连接失败，不能据此判断远端有无相关文件。

> 用户：这部分的record copy 到 /open-record下边

当时结果：先建立本地 `open-record/README.md` 整理副本，当时尚未推送。随后用户要求在 GitHub README 建立 Topic 索引，本页即按该要求整合；历史部署记录与本次文档整理分开标明。

### 2026-09-28：不使用统一脚本，手工启动两个实例

> 用户：start-two-openclaw里边的命令行是什么
>
> 用户：如果手工launch 两个openclaw 应该用什么命令呢

操作与决定：根据历史脚本拆分出加载和启动命令。以下命令在 **iMac 的终端**执行，不是在 M1 或 Mac16 上执行；不需要重新安装 OpenClaw。

#### 1. 先检查共享 Ollama

```bash
sudo lsof -nP -iTCP:11434 -sTCP:LISTEN
```

如果没有监听，先启动 Ollama。历史脚本使用的启动方式为：

```bash
sudo -u mac env HOME=/Users/mac OLLAMA_HOST=127.0.0.1:11434 \
  /Applications/Ollama.app/Contents/Resources/ollama serve
```

这是前台运行方式，保持该终端打开，另开终端执行后续命令；如果 Ollama 已在运行，不要重复启动。账户和应用路径依据当时部署，若已改变应先核对。

#### 2. 尚未加载时，分别加载服务

可先用 `sudo launchctl print system/ai.openclaw.imac-existing` 和 `sudo launchctl print system/ai.openclaw.migrated-m1` 检查。只对未加载的服务执行对应命令：

```bash
sudo launchctl bootstrap system /Library/LaunchDaemons/ai.openclaw.imac-existing.plist
sudo launchctl bootstrap system /Library/LaunchDaemons/ai.openclaw.migrated-m1.plist
```

已加载的服务跳过这一步；不要把重复 bootstrap 的报错当成需要重装。

#### 3. 分别启动两个实例

```bash
sudo launchctl kickstart system/ai.openclaw.imac-existing
sudo launchctl kickstart system/ai.openclaw.migrated-m1
```

| 操作 | 效果 |
|---|---|
| `bootstrap` | 将未加载的服务加入系统 launchd |
| `kickstart` | 启动服务，不强制杀掉正在运行的进程 |
| `kickstart -k` | 强制重启目标服务，会打断该实例当前连接 |
| `start-two-openclaw` | 自动准备 Ollama，并对两个实例执行强制重启式启动 |

#### 4. 检查端口

```bash
sudo lsof -nP -iTCP:11434 -iTCP:18789 -iTCP:19789 -sTCP:LISTEN
```

结论与边界：这套步骤是历史部署的手工启动说明，本次仅更新文档，没有远程执行。三个端口监听只表示服务端口已打开，不代表 QQBot 已连接或模型推理成功；完整可用性仍需分别验证。

### 本地文件

部署说明和回滚材料保存在 iMac：

```text
/Users/mac/.migration-backups/dual-openclaw-20260831-safe/
```

统一启动脚本位于：

```text
/usr/local/bin/start-two-openclaw
```

### 证据边界

本页依据本地 Codex 会话记录整理，原始会话标识为：

`019fc22b-18f8-7c00-a1ab-a0b2b12f28bf`

整理日期：2026-09-28；时间按 Asia/Shanghai。只节选本项目用户可见对话与结果，不包含凭据、工具调用或原始命令输出。本项目没有已产出的结果图片，因此未添加占位图。

目标仓库：[Simon-Zhongyuan-Tian/open-record](https://github.com/Simon-Zhongyuan-Tian/open-record)。上述部署路径均在 iMac 上，不代表脚本已经包含在本地记录目录或 GitHub 仓库中。

[返回主题索引](#index)

---

<a id="topic-02"></a>

## Topic 02 · ChIA-DropBox2 命令参考

保留仓库已有文档：[index.rst](index.rst)。其中包含 `cdb2 CHIADROP`、`SPRITE`、`GAM` 的参数、示例和输出格式。

证据边界：本次仅建立索引，未修改原文档，也未运行或验证其命令；原文未注明记录日期。

[返回主题索引](#index)
