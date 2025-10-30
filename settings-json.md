---
title: mcp-server settings.json For Gemini-cli
description: 
published: 1
date: 2025-09-25T01:28:13.502Z
tags: mcp,gemini,gemini-cli,mcpServers,settings.json 
editor: markdown
dateCreated: 2025-09-25T01:28:13.502Z
---

# mcp-server settings.json For Gemini-cli

[English](/en/settings-json.md) | [Japanese](/ja/settings-json.md) | [Chinese](/settings-json.md)

> **settings.json 存放路径说明：**
>
> 1. **系统路径**：如果希望所有用户或全局环境都能访问该配置文件，可将 `settings.json` 放在操作系统的全局配置目录下。例如：
>    - Windows: `C:\Users\<用户名>\AppData\Roaming\gemini\settings.json`
>    - macOS/Linux: `/home/<用户名>/.config/gemini/settings.json` 或 `/etc/gemini/settings.json`
>    - **影响范围**：全局生效，适用于该操作系统下所有用户或所有项目。
>
> 2. **项目文件夹路径**：推荐将 `settings.json` 放在项目根目录下的 `.gemini` 文件夹中，便于项目成员共享和版本管理。例如：
>    - `your-project/.gemini/settings.json`
>    - 本项目示例：`c:\work\mcp-server\excel-mcp-server\.gemini\settings.json`
>    - **影响范围**：仅对当前项目生效，适合团队协作和本地开发。
>
> 3. **优先级说明**：通常项目文件夹下的 `.gemini/settings.json` 优先级高于系统全局配置，便于本地开发和团队协作时自定义配置。
>
> ---
>
> **settings.json 数据结构说明：**
>
> - 根对象为一个 JSON 对象，包含 `mcpServers` 字段。
> - `mcpServers` 是一个对象，key 为各 mcp-server 服务的唯一标识（如 "github"、"excel" 等）。
> - 每个服务的 value 是一个对象，包含该服务的连接方式、参数、命令、环境变量等配置项。
> - 具体字段和含义可参考下方各服务的配置示例。

```json
{
  "mcpServers": {
    // 这里配置所有可用的 mcp-server 服务，每个服务以唯一 key 标识
    // 例如 "github"、"awslabs.aws-documentation-mcp-server"、"excel" 等
    // 每个服务下有不同的连接方式和参数配置
  }
}
```

### 1. github-mcp-server

```json
    "github": {
      // github-mcp-server 的 HTTP 接口地址
      "httpUrl": "https://api.githubcopilot.com/mcp/",
      // 认证信息，需替换为实际的访问令牌
      "headers": {
        "Authorization": "<public-access-token>"
      },
      // 请求超时时间（毫秒），此处为 5 秒
      "timeout": 5000
    }
```

### 2. awslabs.aws-documentation-mcp-server

```json
    "awslabs.aws-documentation-mcp-server": {
      // 启动该服务的命令，使用 uvx 工具
      "command": "uvx",
      // 启动参数，指定使用最新版本
      "args": ["awslabs.aws-documentation-mcp-server@latest"],
      // 环境变量配置，这里设置日志级别为 ERROR
      "env": {
        "FASTMCP_LOG_LEVEL": "ERROR"
      },
      // 是否禁用该服务，false 表示启用
      "disabled": false,
      // 自动批准的操作列表，默认为空数组
      "autoApprove": []
    }
```

### 3. excel-mcp-server

```json
    "excel": {
      // 启动 excel-mcp-server 的命令
      "command": "uvx",
      // 启动参数，指定以 stdio 模式运行
      "args": ["excel-mcp-server", "stdio"]
    }
```
