🦞 Clawdbot 部署实战手册 (WSL 深度适配版)
==============================
> **作者注**：由于国内网络环境及 Node.js 22 的特殊性（fetch 忽略系统代理），本指南重点解决了 WSL2 镜像网络模式下的代理穿透、权限锁定及持久化运行问题。
> 这里我没有多的vps就直接本地部署了，服务器也差不多的，有问题丢给ai就行。

---

## 一、 准备工作

* **环境**：WSL2 (Ubuntu 24.04) 建议开启 **镜像模式 (Mirror Mode)**。
* **中转 API**：需支持 OpenAI 兼容格式及 Function Calling 能力（推荐使用 Gemini-3-Flash，速度快且支持并发）。
* **代理软件**：Windows 端代理（如 Clash/V2RayN），需开启 **Allow LAN（允许局域网连接）**，端口假设为 `7897`。
* **Telegram**：通过 `@BotFather` 申请到的 `botToken`。

---

## 二、 核心安装步骤

### 1. 安装 Node.js 22+ (解决版本冲突)

**踩坑总结**：直接用 `apt` 安装通常是低版本。即使使用官方 NodeSource 脚本，若系统中存在 NVM，`apt` 安装的版本也会被 NVM 路径覆盖。

#### 推荐方案（NVM 用户）：

```bash
nvm install 22
nvm alias default 22
nvm use 22
node -v  # 必须确保输出 v22.x.x
```

#### 原生安装（非 NVM 用户）：

```bash
curl -fsSL https://deb.nodesource.com/setup_22.x | sudo -E bash -
sudo apt-get install -y nodejs
```

### 2. 全局安装 Clawdbot

```bash
npm install -g clawdbot@latest
clawdbot --version  # 验证版本，应为 2026.1.24-3 或更高
```

---

## 三、 网络与代理配置 (关键突破口)

**难题**：Node.js 22 原生 `fetch` (undici) 会顽固地忽略系统环境变量 `HTTP_PROXY`，导致 `TypeError: fetch failed`。

### 1. 环境验证

在 WSL 中运行以下命令，确保能 Ping 通代理（镜像模式下使用 127.0.0.1）：

```bash
export http_proxy="http://127.0.0.1:7897"
curl -I https://api.telegram.org  # 必须返回 HTTP/2 200 或 302
```

### 2. 使用 Proxychains4 强制转发 (终极方案)

这是解决 Node.js 连不上 Telegram 最稳定的方法，它能从 TCP 层强制拦截请求。

```bash
sudo apt install proxychains4
sudo nano /etc/proxychains4.conf
# 修改文件最后一行 [ProxyList]，将默认的 socks4 改为你的代理：
# http 127.0.0.1 7897
```

---

## 四、 配置文件设计 (`~/.clawdbot/clawdbot.json`)

**踩坑总结**：

1. **权限问题**：`workspace` 绝对不能写 `/wangwang`（普通用户无根目录写入权），必须指向用户目录。
2. **API 协议**：`api` 字段必须是 `openai-completions`，填 `openai-chat` 会导致启动失败。

### 方案一：中转站的配置

```json
{
  "gateway": {
    "mode": "local",
    "bind": "loopback",
    "port": 18789
  },
  "agents": {
    "defaults": {
      "model": { "primary": "gemini/gemini-3-flash" },
      "elevatedDefault": "full",
      "workspace": "/home/austoin/wangwang",
      "compaction": { "mode": "safeguard" },
      "maxConcurrent": 4,
      "subagents": { "maxConcurrent": 8 }
    }
  },
  "models": {
    "mode": "merge",
    "providers": {
      "gemini": {
        "baseUrl": "https://你的中转API/v1",
        "apiKey": "你的APIKey",
        "api": "openai-completions",
        "models": [
          { "id": "gemini-3-flash", "name": "gemini-3-flash" }
        ]
      }
    }
  },
  "channels": {
    "telegram": { "botToken": "你的TG-Token" }
  },
  "plugins": {
    "entries": { "telegram": { "enabled": true } }
  }
}
```

### 方案二：Anthropic 官方 API的配置

```json
{
  "gateway": {
    "mode": "local",
    "bind": "loopback",
    "port": 18789
  },
  "env": {
    "ANTHROPIC_API_KEY": "sk-ant-你的密钥"
  },
  "agents": {
    "defaults": {
      "model": {
        "primary": "anthropic/claude-sonnet-4-5-20261022"
      }
    }
  },
  "channels": {
    "telegram": {
      "enabled": true,
      "botToken": "你的Bot Token",
      "dmPolicy": "pairing"
    }
  }
}
```

---

## 五、 启动、配对与自启

### 1. 启动测试

使用 `proxychains4` 启动：

```bash
proxychains4 -q clawdbot gateway --verbose
```

**日志验证**：

* 看到 `[telegram] [default] starting provider` 说明网络已通。
* 接收消息后看到 `embedded run start` 说明 API 已通。

### 2. 账号配对

1. 在 Telegram 中给你的 Bot 发送 `/start`。
2. Bot 会回复一个配对码（如 `MJRC8QDH`）。
3. 在新终端执行：
```bash
clawdbot pairing approve telegram [你的配对码]
```

### 3. 设置开机自启

#### 方案 A：PM2 进程管理 (WSL 用户首选)

```bash
npm install -g pm2
# 创建脚本 start_clawd.sh
echo "proxychains4 -q clawdbot gateway --verbose" > ~/start_clawd.sh
chmod +x ~/start_clawd.sh
# 启动
pm2 start ~/start_clawd.sh --name "clawdbot"
# 验证是否运行
pm2 list
# 最关键的一步：保存列表（WSL重启后自动恢复）
pm2 save
```

**验证与启动后检查**：

运行 `pm2 list` 后，你应该能看到一行 name 为 `clawdbot` 且 status 为 `online` 的记录。

**PM2 常用管理命令**：

```bash
# 查看实时日志
pm2 logs clawdbot

# 重启服务（修改配置后使用）
pm2 restart clawdbot

# 停止服务
pm2 stop clawdbot

# 查看所有进程
pm2 list

# 删除所有进程
pm2 delete all
```

#### 方案 B：Systemd 系统服务 (云服务器首选)

在 WSL 中，建议使用 `screen` 或 `tmux` 挂起进程。如果在标准 Linux 服务器上，创建 `/etc/systemd/system/clawdbot.service`：

```ini
[Unit]
Description=Clawdbot Service
After=network.target

[Service]
Type=simple
User=austoin
# 注意：Environment 中的 PATH 必须包含你实际的 node bin 路径
Environment="PATH=/home/austoin/.nvm/versions/node/v22.22.0/bin:/usr/local/bin:/usr/bin:/bin"
Environment="HOME=/home/austoin"
ExecStart=/usr/bin/proxychains4 -q /home/austoin/.nvm/versions/node/v22.22.0/bin/clawdbot gateway --verbose
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

_注意：`ExecStart` 路径需通过 `which proxychains4` 和 `which clawdbot` 确认。_

---

## 六、 维护常用命令

* **查看健康状态**：`clawdbot doctor`
* **查看所有状态**：`clawdbot status --all`
* **实时查看日志**：`pm2 logs clawdbot` 或 `journalctl -u clawdbot -f`
* **实时日志**：运行 `gateway --verbose` 或查看 `/tmp/clawdbot/` 下的日志文件。
* **强制重装 Node 版本**：
```bash
sudo apt-get remove --purge nodejs npm -y && sudo apt autoremove
# 重新通过 nvm install 22 安装
```

---

## 下一步建议

既然机器人已经能说话了，你可以尝试让它集成 **Exa** 进行联网搜索。只需在 `plugins` 中添加 Exa 的 API Key 即可。

你可以试着给 Bot 发一句"你好"，如果它回复了，就可以尝试让它帮你写代码、查文档，甚至通过集成 Exa 插件来增强它的搜索能力了。
