---
name: wechat-article-downloader
description: Download WeChat official account articles in multiple formats (HTML, PDF, Word, Markdown, TXT, MHTML), including single article download, collection (appmsgalbum) download, local bulk workflows, and metadata export. Use this skill when users ask to download WeChat articles, download public account collections, batch pull account content, or export article data.
---

# WeChat Article Downloader

This skill helps you download articles from WeChat official accounts using an MCP server that interfaces with a WeChat article download tool.

## Prerequisites

The skill uses local MCP endpoint `http://127.0.0.1:4545/mcp` for full functionality.

If local MCP is unavailable, you may fallback to `https://changfengbox.top/api/mcp` for remote-supported tools.

Local MCP tools:
- `single_article_download` - Download a single article
- `get_public_account_id` - Get public account credentials
- `batch_download_articles` - Batch download all articles from an account
- `export_article_data` - Export article metadata to CSV

Remote fallback MCP (`https://changfengbox.top/api/mcp`) currently supports:
- Methods: `initialize`, `tools/list`, `tools/call`
- Tools in `tools/call` `name`:
  - `wechat` - Download a single article
  - `wechat_collection` - Download a public account collection (`appmsgalbum`)

Important: remote fallback does **not** expose local tool names like `single_article_download`, `batch_download_articles`, `get_public_account_id`, or `export_article_data`.

## Workflow

### 1. Check Local MCP Server Status

Before using most functionality, verify local MCP is running:

```python
import requests

LOCAL_MCP_ENDPOINT = "http://127.0.0.1:4545/mcp"

def check_local_mcp():
    response = requests.post(
        LOCAL_MCP_ENDPOINT,
        json={"jsonrpc": "2.0", "method": "initialize", "id": 1},
        headers={"Content-Type": "application/json"},
        timeout=3,
    )
    if response.status_code != 200:
        raise RuntimeError("Local MCP is not available")
    return LOCAL_MCP_ENDPOINT

local_mcp_endpoint = check_local_mcp()
```

If local MCP is inaccessible, inform the user they need to:
1. Open the WeChat article download tool
2. Check the "启动MCP" checkbox to start the MCP service
3. Wait for the confirmation message showing the service is running on port 4545

### 2. Single Article Download

Use local tool first. If local MCP fails, fallback to remote `wechat` tool.

```python
import requests

LOCAL_MCP_ENDPOINT = "http://127.0.0.1:4545/mcp"
FALLBACK_DOWNLOAD_MCP_ENDPOINT = "https://changfengbox.top/api/mcp"

def call_tool(endpoint, tool_name, arguments, req_id=2):
    payload = {
        "jsonrpc": "2.0",
        "method": "tools/call",
        "id": req_id,
        "params": {
            "name": tool_name,
            "arguments": arguments,
        }
    }

    response = requests.post(
        endpoint,
        json=payload,
        headers={"Content-Type": "application/json"},
        timeout=10,
    )
    response.raise_for_status()
    return response.json()

def download_single_article(url):
    try:
        return call_tool(
            LOCAL_MCP_ENDPOINT,
            "single_article_download",
            {"url": url},
            req_id=2,
        )
    except Exception:
        remote_config = {
            "保存离线网页": True,
            "HTML": True,
            "MD": True,
            "PDF": False,
            "WORD": False,
            "TXT": False,
            "MHTML": False,
            "文件开头添加日期": True,
        }
        return call_tool(
            FALLBACK_DOWNLOAD_MCP_ENDPOINT,
            "wechat",
            {"url": url, "config": remote_config},
            req_id=3,
        )

# Example usage
article_url = "https://mp.weixin.qq.com/s/xxxxx"
result = download_single_article(article_url)
print(result)
```

The article will be downloaded to the tool's default download directory in the formats configured in the tool (HTML, PDF, Word, Markdown, MHTML, etc.).

Remote `wechat` config can use these keys:
- `保存离线网页`
- `HTML`
- `MD`
- `PDF`
- `WORD`
- `TXT`
- `MHTML`
- `文件开头添加日期`

If local `127.0.0.1:4545` is unavailable, this step should transparently use `https://changfengbox.top/api/mcp` with tool name `wechat`.

### 3. Download Collection (`appmsgalbum`)

For collection URLs, call remote `wechat_collection` via `tools/call`:

```python
def download_collection(collection_url):
    payload = {
        "jsonrpc": "2.0",
        "method": "tools/call",
        "id": 4,
        "params": {
            "name": "wechat_collection",
            "arguments": {
                "url": collection_url,
            },
        },
    }

    response = requests.post(
        "https://changfengbox.top/api/mcp",
        json=payload,
        headers={"Content-Type": "application/json"},
        timeout=20,
    )
    response.raise_for_status()
    return response.json()
```

### 4. Get Public Account Credentials (Local Only)

Before batch downloading, you need to obtain the public account credentials. This triggers a process where:
1. The tool extracts the account ID from a sample article URL
2. Generates a special link that needs to be opened in WeChat desktop client
3. Automatically captures the authentication credentials when you open the link

```python
def get_account_credentials():
    payload = {
        "jsonrpc": "2.0",
        "method": "tools/call",
        "id": 3,
        "params": {
            "name": "get_public_account_id",
            "arguments": {}
        }
    }
    
    response = requests.post('http://127.0.0.1:4545/mcp',
                            json=payload,
                            headers={"Content-Type": "application/json"})
    
    result = response.json()
    return result

result = get_account_credentials()
print(result)
```

After calling this, the user needs to:
1. Copy the generated link from the tool's log window
2. Open it in WeChat desktop client (not browser)
3. Wait for the tool to automatically capture the credentials
4. Look for "获取密钥成功" (credentials obtained successfully) message

### 5. Batch Download Articles (Local Only)

Once credentials are obtained, you can batch download all articles from the public account:

```python
def batch_download_articles():
    payload = {
        "jsonrpc": "2.0",
        "method": "tools/call",
        "id": 4,
        "params": {
            "name": "batch_download_articles",
            "arguments": {}
        }
    }
    
    response = requests.post('http://127.0.0.1:4545/mcp',
                            json=payload,
                            headers={"Content-Type": "application/json"})
    
    result = response.json()
    return result

result = batch_download_articles()
print(result)
```

The batch download will:
- Download articles according to the date range and filters configured in the tool
- Save articles in multiple formats (HTML, PDF, Word, Markdown, MHTML as configured)
- Organize files by public account name
- Show progress in the tool's log window

### 6. Export Article Data (Local Only)

To export article metadata (titles, URLs, publish dates, read counts, like counts, etc.) to CSV:

```python
def export_article_data():
    payload = {
        "jsonrpc": "2.0",
        "method": "tools/call",
        "id": 5,
        "params": {
            "name": "export_article_data",
            "arguments": {}
        }
    }
    
    response = requests.post('http://127.0.0.1:4545/mcp',
                            json=payload,
                            headers={"Content-Type": "application/json"})
    
    result = response.json()
    return result

result = export_article_data()
print(result)
```

This exports a CSV file containing article metadata to the download directory.

## Common Workflows

### Workflow 1: Download a Single Article

User says: "Download this WeChat article: https://mp.weixin.qq.com/s/xxxxx"

1. Check local MCP server status
2. If local is running, call `single_article_download` with the URL
3. If local is unavailable, call remote `wechat` with `url + config`
4. Inform user the download has started and where files will be saved

### Workflow 2: Download a Collection (`appmsgalbum`)

User says: "Download this WeChat collection/appmsgalbum"

1. Call remote `tools/call` with `name="wechat_collection"`
2. Pass the collection URL in `arguments`
3. If needed, validate tool arguments by calling `tools/list` first
4. Inform user where files are saved

### Workflow 3: Batch Download from a Public Account (Local Only)

User says: "Download all articles from this public account"

1. Check local MCP server status
2. Ensure user has provided a sample article URL from the account
3. Call `get_public_account_id` and explain the user needs to open the generated link in WeChat
4. Wait for user confirmation that credentials were obtained
5. Call `batch_download_articles` to start the bulk download
6. Monitor progress through the tool's interface

### Workflow 4: Export Article Metadata (Local Only)

User says: "Export article data to CSV" or "Get article statistics"

1. Check local MCP server status
2. Ensure credentials are already obtained (if not, guide through credential process)
3. Call `export_article_data`
4. Inform user where the CSV file is saved

## Error Handling

Common issues and solutions:

**Local MCP unavailable during single download**: Automatically switch to `https://changfengbox.top/api/mcp` and retry with `name="wechat"`.

**Remote tool name mismatch**: Do not use local names on remote fallback. Use only `wechat` or `wechat_collection`.

**Local MCP unavailable for credentials/batch/export**: These are local-only workflows. User needs to start the WeChat download tool and enable MCP service.

**Credentials not obtained**: User needs to complete the credential acquisition process by opening the generated link in WeChat desktop client

**Download fails**: Check the tool's log window for specific error messages. Common causes:
- Invalid article URL format
- Network connectivity issues
- WeChat rate limiting

## Output Locations

All downloads are saved to the tool's configured download directory (default: `下载/` folder in the tool's directory), organized by:
- Public account name (for batch downloads)
- Article title
- Format (HTML, PDF, Word, Markdown, MHTML, etc.)

The tool's interface shows the exact file paths in the log window as downloads complete.
