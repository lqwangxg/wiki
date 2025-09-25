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

> **settings.json Storage Path Description:**
>
> 1. **System Path**: If you want all users or the global environment to access this configuration file, you can place `settings.json` in the operating system's global configuration directory. For example:
>    - Windows: `C:\Users\<username>\AppData\Roaming\gemini\settings.json`
>    - macOS/Linux: `/home/<username>/.config/gemini/settings.json` or `/etc/gemini/settings.json`
>    - **Scope**: Globally effective, applicable to all users or all projects under that operating system.
>
> 2. **Project Folder Path**: It is recommended to place `settings.json` in the `.gemini` folder under the project root directory, which facilitates sharing and version management among project members. For example:
>    - `your-project/.gemini/settings.json`
>    - This project example: `c:\work\mcp-server\excel-mcp-server\.gemini\settings.json`
>    - **Scope**: Only effective for the current project, suitable for team collaboration and local development.
>
> 3. **Priority Description**: Generally, `.gemini/settings.json` under the project folder has higher priority than the system's global configuration, which facilitates custom configuration during local development and team collaboration.
>
> ---
>
> **settings.json Data Structure Description:**
>
> - The root object is a JSON object containing the `mcpServers` field.
> - `mcpServers` is an object where the key is the unique identifier of each mcp-server service (e.g., "github", "excel", etc.).
> - The value of each service is an object containing configuration items such as the service's connection method, parameters, commands, and environment variables.
> - For specific fields and meanings, please refer to the configuration examples for each service below.

```json
{
  "mcpServers": {
    // All available mcp-server services are configured here, each service identified by a unique key
    // For example, "github", "awslabs.aws-documentation-mcp-server", "excel", etc.
    // Each service has different connection methods and parameter configurations
  }
}
```

### 1. github-mcp-server

```json
    "github": {
      // HTTP interface address of github-mcp-server
      "httpUrl": "https://api.githubcopilot.com/mcp/",
      // Authentication information, needs to be replaced with an actual access token
      "headers": {
        "Authorization": "<public-access-token>"
      },
      // Request timeout in milliseconds, here it is 5 seconds
      "timeout": 5000
    }
```

### 2. awslabs.aws-documentation-mcp-server

```json
    "awslabs.aws-documentation-mcp-server": {
      // Command to start this service, using the uvx tool
      "command": "uvx",
      // Startup parameters, specifying to use the latest version
      "args": ["awslabs.aws-documentation-mcp-server@latest"],
      // Environment variable configuration, here the log level is set to ERROR
      "env": {
        "FASTMCP_LOG_LEVEL": "ERROR"
      },
      // Whether to disable this service, false means enabled
      "disabled": false,
      // List of automatically approved operations, defaults to an empty array
      "autoApprove": []
    }
```

### 3. excel-mcp-server

```json
    "excel": {
      // Command to start excel-mcp-server
      "command": "uvx",
      // Startup parameters, specifying to run in stdio mode
      "args": ["excel-mcp-server", "stdio"]
    }
```