# CLIProxyAPI 服务工具（CPA）

> 本地 `cli-proxy-api`（v7.2.130）的一体化启停与管理脚本，支持通过 `.env` 文件保存 Provider API Key，避免密钥明文写入 `config.yaml`。

本工具是一个 shell 包装脚本 `cpa`，用于管理后台运行的本机 CLIProxyAPI 代理服务，并提供常用的启停、状态查看、日志与信息查询功能。

---

## 目录

- [功能特性](#功能特性)
- [文件布局](#文件布局)
- [快速开始](#快速开始)
- [命令说明](#命令说明)
- [使用 .env 管理 API Key](#使用-env-管理-api-key)
- [配置文件说明](#配置文件说明)
- [环境变量](#环境变量)
- [常见问题](#常见问题)
- [安全建议](#安全建议)

---

## 功能特性

- `nohup` 后台启动 `cli-proxy-api`，自动管理 PID 文件
- 启动/停止/状态/信息/日志 五个常用子命令
- 自动探测二进制与 `config.yaml` 路径
- **`.env` 支持**：启动时自动加载配置文件同目录的 `.env`，将 `config.yaml` 中的 `${VAR_NAME}` 占位符渲染为真实密钥，避免明文
- 未找到 `.env` 时给出配置提示，但**仍按默认方式加载**，不影响正常使用
- 停止时自动清理渲染产生的临时配置文件

---

## 文件布局

```
CPA/
├── cpa                 # 主脚本（包装脚本）
├── cli-proxy-api       # CPA 二进制（v7.2.130）
├── config.yaml         # 配置文件（可含 ${VAR} 占位符）
├── .env                # 密钥文件（与 config.yaml 同目录，不入库）
├── .env.example        # 环境变量示例（模板）
├── oauth/              # OAuth 凭证目录
└── README.md           # 本说明
```

> `.env` 必须与 `config.yaml` 放在**同一目录**，脚本按此规则探测。

---

## 快速开始

```bash
# 1. 首次配置：复制示例并填写密钥
cp .env.example .env
vim .env        # 填写真实 API Key

# 2. 编辑 config.yaml，把 API Key 换成占位符
#    api-key: "${OLLAMA_API_KEY}"

# 3. 启动服务
cpa on

# 4. 查看状态
cpa status
```

---

## 命令说明

| 命令 | 功能 |
|------|------|
| `cpa on` | 启动服务（后台运行），打印关键信息 |
| `cpa off` | 停止服务，清理临时配置 |
| `cpa status` | 查看运行状态与关键信息 |
| `cpa info` | 只打印关键信息，不启停 |
| `cpa log` | 实时查看日志（Ctrl+C 退出） |
| `cpa help` | 显示帮助 |

`cpa on` 输出示例：

```text
✔ 已加载环境变量: /usr/local/bin/CLIProxyAPI/.env
✔ 已生成临时渲染配置: /var/folders/.../cpa-config.xxxxxx.yaml
✔ cli-proxy-api 已启动 (PID 12345)
```

---

## 使用 .env 管理 API Key

### 原理

1. 在 `config.yaml` 中把 API Key 写成占位符：`api-key: "${OPENCODE_GO_API_KEY_1}"`
2. 把真实值写入同目录的 `.env`：

   ```bash
   OPENCODE_GO_API_KEY_1=sk-xxxx
   ```

3. `cpa on` 启动时：
   - 自动探测并加载 `.env`
   - 用 Python3 把 `${VAR}` 占位符展开为环境变量值
   - 生成临时配置 `/var/folders/.../cpa-config.xxxxxx.yaml`
   - 用临时配置启动 CPA
4. `cpa off` 停止时自动清理临时配置

### 找不到 .env 时

`cpa on/info/status` 会提示：

```text
.env           : 未找到（当前按 config.yaml 原样加载，API Key 为明文）
                提示：为避免明文，可把 API Key 写成 ${VAR_NAME} 占位符，
                并将真实值写入 <config 目录>/.env（与 config.yaml 同目录）
                例如 .env 内容: OPENCODE_GO_API_KEY_1=sk-xxxx
```

此时脚本仍按默认方式读取 `config.yaml` 原样启动，不会因缺少 `.env` 而失败。

---

## 配置文件说明

`config.yaml` 关键字段：

| 字段 | 说明 |
|------|------|
| `host` / `port` | 监听地址与端口（默认 `127.0.0.1:8317`） |
| `api-keys` | 下游客户端访问 CPA 的本地访问令牌 |
| `remote-management.secret-key` | WebUI 管理密码（首次启动自动哈希） |
| `auth-dir` | OAuth/凭证目录 |
| `routing.strategy` | 负载均衡策略：`round-robin` / `weighted-round-robin` / `fill-first` |
| `routing.session-affinity` | 同一会话绑定同一 key |
| `providers` | 上游 Provider（`claude-api-key`、`openai-compatibility` 等），API Key 可用 `${VAR}` 占位符 |

Provider 模型可配置 `max-context-length`，避免下游客户端一律显示 200k。

---

## 环境变量

### 脚本可用环境变量

| 变量 | 默认 | 说明 |
|------|------|------|
| `CPA_BIN` | 自动探测 | 二进制路径 |
| `CPA_CONFIG` | 自动探测 | 配置文件路径 |
| `CPA_ENV` | 自动探测（config 同目录） | `.env` 文件路径 |
| `CPA_LOG` | `~/Library/Logs/cliproxyapi.log` | 日志文件路径 |
| `CPA_PIDFILE` | `${TMPDIR:-/tmp}/cpa.pid` | PID 文件路径 |

### .env 中的密钥变量（示例）

见 `.env.example`：

```bash
# Ollama 本地模型
OLLAMA_API_KEY=your-ollama-key

# opencode go 账号 1
OPENCODE_GO_API_KEY_1=sk-your-key-1

# opencode go 账号 2
OPENCODE_GO_API_KEY_2=sk-your-key-2
```

---

## 常见问题

**Q1：`cpa status` 报错 / 显示启动异常？**
先看日志：`cpa log`。若日志显示 `failed to read config file: open ... no such file`，说明临时配置路径异常，请确认 `.env` 变量名与 `config.yaml` 占位符一致。

**Q2：修改 `.env` 后需要重启吗？**
需要。`cpa off` 后再 `cpa on`，脚本会重新加载 `.env` 并重新渲染临时配置。

**Q3：`mktemp` 相关问题？**
macOS 的 `mktemp` 要求 `X` 在文件名末尾，脚本已处理（生成 `cpa-config.XXXXXX` 后重命名 `.yaml`）。

**Q4：临时配置文件在哪里？**
`${TMPDIR:-/tmp}/cpa-config.xxxxxx.yaml`，停止时自动删除。

---

## 安全建议

- `.env` 权限建议 `600`：`chmod 600 .env`
- 不要把 `.env` 提交到 Git 或备份到不受控位置
- 临时配置只在运行期间存在于系统临时目录，停止后自动清理
- 定期轮换上游 API Key
