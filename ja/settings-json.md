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

> **settings.json 保存パスの説明：**
>
> 1. **システムパス**：すべてのユーザーまたはグローバル環境でこの設定ファイルにアクセスできるようにしたい場合は、`settings.json` をオペレーティングシステムのグローバル設定ディレクトリに配置できます。例：
>    - Windows: `C:\Users\<ユーザー名>\AppData\Roaming\gemini\settings.json`
>    - macOS/Linux: `/home/<ユーザー名>/.config/gemini/settings.json` または `/etc/gemini/settings.json`
>    - **影響範囲**：グローバルに有効で、そのオペレーティングシステム下のすべてのユーザーまたはすべてのプロジェクトに適用されます。
>
> 2. **プロジェクトフォルダパス**：プロジェクトメンバー間での共有とバージョン管理を容易にするため、`settings.json` をプロジェクトのルートディレクトリ下の `.gemini` フォルダに配置することをお勧めします。例：
>    - `your-project/.gemini/settings.json`
>    - このプロジェクトの例：`c:\work\mcp-server\excel-mcp-server\.gemini\settings.json`
>    - **影響範囲**：現在のプロジェクトのみに有効で、チームコラボレーションやローカル開発に適しています。
>
> 3. **優先順位の説明**：通常、プロジェクトフォルダ下の `.gemini/settings.json` はシステムグローバル設定よりも優先度が高く、ローカル開発やチームコラボレーション時のカスタム設定を容易にします。
>
> ---
>
> **settings.json データ構造の説明：**
>
> - ルートオブジェクトは `mcpServers` フィールドを含む JSON オブジェクトです。
> - `mcpServers` はオブジェクトで、キーは各 mcp-server サービスの一意の識別子（例：「github」、「excel」など）です。
> - 各サービスの値は、そのサービスの接続方法、パラメータ、コマンド、環境変数などの設定項目を含むオブジェクトです。
> - 特定のフィールドと意味については、以下の各サービスの設定例を参照してください。

```json
{
  "mcpServers": {
    // ここに利用可能なすべての mcp-server サービスを設定します。各サービスは一意のキーで識別されます。
    // 例：「github」、「awslabs.aws-documentation-mcp-server」、「excel」など
    // 各サービスには異なる接続方法とパラメータ設定があります。
  }
}
```

### 1. github-mcp-server

```json
    "github": {
      // github-mcp-server の HTTP インターフェースアドレス
      "httpUrl": "https://api.githubcopilot.com/mcp/",
      // 認証情報。実際のアクセストークンに置き換える必要があります。
      "headers": {
        "Authorization": "<public-access-token>"
      },
      // リクエストのタイムアウト時間（ミリ秒）。ここでは 5 秒です。
      "timeout": 5000
    }
```

### 2. awslabs.aws-documentation-mcp-server

```json
    "awslabs.aws-documentation-mcp-server": {
      // このサービスを起動するコマンド。uvx ツールを使用します。
      "command": "uvx",
      // 起動パラメータ。最新バージョンを使用することを指定します。
      "args": ["awslabs.aws-documentation-mcp-server@latest"],
      // 環境変数設定。ここではログレベルを ERROR に設定しています。
      "env": {
        "FASTMCP_LOG_LEVEL": "ERROR"
      },
      // このサービスを無効にするかどうか。false は有効を意味します。
      "disabled": false,
      // 自動承認される操作のリスト。デフォルトは空の配列です。
      "autoApprove": []
    }
```

### 3. excel-mcp-server

```json
    "excel": {
      // excel-mcp-server を起動するコマンド
      "command": "uvx",
      // 起動パラメータ。stdio モードで実行することを指定します。
      "args": ["excel-mcp-server", "stdio"]
    }
```