# CPA

> CLIProxyAPI service wrapper — elegantly start / stop `cli-proxy-api` / `cliproxyapi` in the background.
> CLIProxyAPI 启停包装脚本：一键后台启停 cli-proxy-api / cliproxyapi，并打印关键信息。

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

`cpa` is a small bash wrapper around [CLIProxyAPI](https://github.com/router-for-me/CLIProxyAPI) that starts the proxy in the background (no terminal occupied), stops it cleanly, and prints the key info you need — binary path, config path, WebUI address, and which config field holds the management password.

> ⚠️ This is an independent community wrapper, **not** affiliated with the official CLIProxyAPI project.

## Features / 功能

- `cpa on` — start in background (nohup) + print key info
- `cpa off` — stop (graceful SIGTERM, force-kill fallback)
- `cpa status` / `cpa info` / `cpa log` / `cpa help`
- Auto-detect the binary & config file (PATH, brew opt, common locations)
- PID-file based process management, with `pkill` fallback
- Never prints secret values — only shows the management key's *status* (set / empty / hashed)

## Install / 安装

Dependencies: `bash`, `python3` (only used to parse the port & key status from config).

```bash
# 放到任意位置（示例：与二进制同目录 + 软链接到 PATH）
curl -fsSL https://raw.githubusercontent.com/panando/CPA/main/cpa -o /usr/local/bin/CLIProxyAPI/cpa
chmod +x /usr/local/bin/CLIProxyAPI/cpa
ln -sf /usr/local/bin/CLIProxyAPI/cpa /usr/local/bin/cpa

# 或放到 ~/.local/bin（需确保在 PATH 中）
mkdir -p ~/.local/bin
curl -fsSL https://raw.githubusercontent.com/panando/CPA/main/cpa -o ~/.local/bin/cpa
chmod +x ~/.local/bin/cpa
```

## Usage / 用法

| Command | Description |
| --- | --- |
| `cpa on` | Start in background and print key info |
| `cpa off` | Stop the service |
| `cpa status` | Show running status + key info |
| `cpa info` | Print key info only (no start/stop) |
| `cpa log` | Tail the log live (Ctrl+C to exit) |
| `cpa help` | Show help |

## What `cpa on` shows / 启动时提示的信息

```
✔ cli-proxy-api 已启动 (PID 12345)

  ── CLIProxyAPI 关键信息 ──
  二进制       : /usr/local/bin/CLIProxyAPI/cli-proxy-api
  配置文件     : /usr/local/bin/CLIProxyAPI/config.yaml
  WebUI 入口   : http://localhost:8317/management.html
  管理密码字段 : remote-management.secret-key
  密码当前状态 : 已设置（bcrypt 哈希，原明文无法找回）
  日志文件     : /Users/you/Library/Logs/cliproxyapi.log
```

**About the management password / 关于管理密码：**
- The WebUI login password is the `remote-management.secret-key` field in `config.yaml` — **not** the `api-keys` (`sk-*`) entries.
- If `secret-key` is empty, the Management API / WebUI is disabled entirely (`/management.html` returns 404).
- Once the service has started, the plaintext key is automatically **hashed (bcrypt)** back into the config file — the original plaintext is **unrecoverable**. To reset: edit `config.yaml` with a new plaintext key, then `cpa off` → `cpa on` (it will re-hash on startup).

## Configuration / 环境变量

| Variable | Default |
| --- | --- |
| `CPA_BIN` | auto-detect |
| `CPA_CONFIG` | auto-detect |
| `CPA_LOG` | `~/Library/Logs/cliproxyapi.log` (Linux: `$HOME/.cache/cliproxyapi.log`) |
| `CPA_PIDFILE` | `${TMPDIR:-/tmp}/cpa.pid` |

Binary detection order: `$CPA_BIN` → `cli-proxy-api` / `cliproxyapi` on PATH → `/opt/homebrew/opt/cliproxyapi/bin/cliproxyapi` → common brew locations.

Config detection order: `$CPA_CONFIG` → `config.yaml` next to the detected binary → `~/.cli-proxy-api/config.yaml` → brew `cliproxyapi.conf`.

## How it works / 工作原理

- `cpa on`: detect bin & config → `nohup "$bin" -config "$config" >>"$LOG" 2>&1 &` → save PID → verify the process survived → print key info.
- `cpa off`: read PID file → `SIGTERM` → wait up to ~6s → `SIGKILL` if unresponsive → `pkill` fallback by process name (case-sensitive, so it never kills the wrapper itself).

## FAQ / 常见问题

**管理密码忘了怎么办？**
Secret-key 一旦被哈希，明文无法找回。在 `config.yaml` 的 `remote-management.secret-key` 填入新明文 → `cpa off` → `cpa on`，服务启动时会自动哈希写回。

**想开机自启、崩溃自动重启？**
`cpa` 走的是纯 `nohup` 后台方式，不注册开机自启。需要常驻可用 brew services：`brew services start cliproxyapi`；或 systemd / Docker。

**macOS 想用系统托盘？**
官方 `--tray` 模式尚未实现（见 upstream issue）。可用官方桌面端 [EasyCLIProxyAPI](https://github.com/router-for-me/EasyCLIProxyAPI)，或本脚本配合 brew services。

## License / 许可证

[MIT](LICENSE) © panando
