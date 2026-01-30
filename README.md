🦞 Clawdbot 部署实战手册 (WSL 深度适配版)
==============================

> 这里我没有多的vps就直接本地部署了，服务器也差不多的，有问题丢给ai就行。

一、 准备工作
-------

* **环境**：WSL2 (Ubuntu 24.04) 镜像模式（Mirror Mode）。

* **中转 API**：支持 OpenAI 兼容格式、具备 Function Calling 能力。

* **代理软件**：Windows 端代理（如 Clash/V2RayN），需开启 **Allow LAN（允许局域网连接）**，端口建议 `7897`。

* **Telegram**：通过 `@BotFather` 申请到的 `botToken`。

* * *

二、 核心安装步骤
---------

### 1. 安装 Node.js 22+ (解决版本冲突)

**踩坑：** 即使使用官方 NodeSource 脚本，如果系统中存在 NVM，`apt` 安装的版本会被覆盖。

* **推荐方案（NVM 用户）**：
  Bash
  
      nvm install 22
      nvm alias default 22
      nvm use 22
      node -v # 必须确保输出 v22.x.x

* **原生安装（非 NVM 用户）**：
  Bash
  
      curl -fsSL https://deb.nodesource.com/setup_22.x | sudo -E bash -
      sudo apt-get install -y nodejs
  
  

### 2. 全局安装 Clawdbot

Bash
    npm install -g clawdbot@latest
    clawdbot --version # 验证版本

* * *

三、 网络与代理配置 (国内 WSL 用户必看)
------------------------

**难题：** Node.js 22 的原生 `fetch` 会忽略系统环境变量，导致 Telegram API 连接失败（`fetch failed`）。

### 1. 环境验证

在 WSL 中运行以下命令，确保能 Ping 通代理：

Bash
    export http_proxy="http://127.0.0.1:7897"
    curl -I https://api.telegram.org # 必须返回 HTTP/2 200

### 2. 使用 Proxychains4 强制转发

这是解决 Node.js 连不上 Telegram 最稳定的方法：

Bash
    sudo apt install proxychains4
    sudo nano /etc/proxychains4.conf
    # 修改文件最后一行，将 socks4 改为你的代理：
    # http 127.0.0.1 7897

* * *

四、 配置文件设计 (`~/.clawdbot/clawdbot.json`)
---------------------------------------

**踩坑：** 1. `workspace` 路径不能写 `/wangwang`（无根目录权限），必须指向用户目录。2. `api` 字段必须是 `openai-completions`。

##### 中转站的配置：

JSON
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
          "maxConcurrent": 4
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



##### Anthropic 官方 API的配置

JSON

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

---

五、 启动、配对与自启
-----------

### 1. 启动测试

使用 `proxychains4` 启动以穿透网络限制：

Bash
    proxychains4 -q clawdbot gateway --verbose

看到 `[telegram] [default] starting provider` 且无报错即为成功。

### 2. 账号配对

1. 在 Telegram 中给你的 Bot 发送 `/start`。

2. Bot 回复一个配对码（如 `MJRC8QDH`）。

3. 在服务器终端执行：
   Bash
      clawdbot pairing approve telegram MJRC8QDH
   
   

### 3. 设置开机自启 (Systemd)

在 WSL 中，建议使用 `screen` 或 `tmux` 挂起进程。如果在标准 Linux 服务器上，创建 `/etc/systemd/system/clawdbot.service`：

Ini, TOML
    [Service]
    ExecStart=/usr/bin/proxychains4 -q /usr/bin/clawdbot gateway --verbose
    Restart=always
    Environment=HOME=/home/austoin
    User=austoin

_注意：`ExecStart` 路径需通过 `which proxychains4` 和 `which clawdbot` 确认。_

* * *

六、 维护常用命令
---------

* **查看健康状态**：`clawdbot doctor`

* **查看所有状态**：`clawdbot status --all`

* **实时日志**：运行 `gateway --verbose` 或查看 `/tmp/clawdbot/` 下的日志文件。

* * *

**下一步建议：** 你可以试着给 Bot 发一句“你好”，如果它回复了，就可以尝试让它帮你写代码、查文档，甚至通过集成 Exa 插件来增强它的搜索能力了。
