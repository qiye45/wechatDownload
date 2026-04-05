---
name: wechat-article-downloader
description: Download WeChat official account articles in various formats (HTML, PDF, Word, Markdown, MHTML). Use this skill when users want to download WeChat articles, save public account content, batch download articles from a WeChat official account, export article data to CSV, or retrieve article metadata. This skill handles both single article downloads and bulk operations, and can export structured data for analysis.
---

# WeChat Article Downloader

This skill helps you download articles from WeChat official accounts using an MCP server that interfaces with a local WeChat article download tool.

## Prerequisites

The skill requires a local MCP server running at `http://127.0.0.1:4545/mcp`. The server provides four main tools:
- `single_article_download` - Download a single article
- `get_public_account_id` - Get public account credentials
- `batch_download_articles` - Batch download all articles from an account
- `export_article_data` - Export article metadata to CSV

## Workflow

### 1. Check MCP Server Status

Before using any functionality, verify the MCP server is running:

```python
import requests
try:
    response = requests.post('http://127.0.0.1:4545/mcp', 
                            json={"jsonrpc": "2.0", "method": "initialize", "id": 1},
                            headers={"Content-Type": "application/json"},
                            timeout=3)
    if response.status_code == 200:
        print("MCP server is running")
    else:
        print("MCP server returned unexpected status")
except:
    print("MCP server is not accessible. Please start the WeChat download tool and enable MCP service (check the '启动MCP' checkbox)")
```

If the server is not accessible, inform the user they need to:
1. Open the WeChat article download tool
2. Check the "启动MCP" checkbox to start the MCP service
3. Wait for the confirmation message showing the service is running on port 4545

### 2. Single Article Download

To download a single WeChat article, call the `single_article_download` tool with the article URL:

```python
import requests
import json

def download_single_article(url):
    payload = {
        "jsonrpc": "2.0",
        "method": "tools/call",
        "id": 2,
        "params": {
            "name": "single_article_download",
            "arguments": {"url": url}
        }
    }
    
    response = requests.post('http://127.0.0.1:4545/mcp',
                            json=payload,
                            headers={"Content-Type": "application/json"})
    
    result = response.json()
    return result

# Example usage
article_url = "https://mp.weixin.qq.com/s/xxxxx"
result = download_single_article(article_url)
print(result)
```

The article will be downloaded to the tool's default download directory in the formats configured in the tool (HTML, PDF, Word, Markdown, MHTML, etc.).

### 3. Get Public Account Credentials

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

### 4. Batch Download Articles

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

### 5. Export Article Data

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

1. Check MCP server status
2. If running, call `single_article_download` with the URL
3. Inform user the download has started and where files will be saved

### Workflow 2: Batch Download from a Public Account

User says: "Download all articles from this public account"

1. Check MCP server status
2. Ensure user has provided a sample article URL from the account
3. Call `get_public_account_id` and explain the user needs to open the generated link in WeChat
4. Wait for user confirmation that credentials were obtained
5. Call `batch_download_articles` to start the bulk download
6. Monitor progress through the tool's interface

### Workflow 3: Export Article Metadata

User says: "Export article data to CSV" or "Get article statistics"

1. Check MCP server status
2. Ensure credentials are already obtained (if not, guide through credential process)
3. Call `export_article_data`
4. Inform user where the CSV file is saved

## Error Handling

Common issues and solutions:

**MCP server not accessible**: User needs to start the WeChat download tool and enable MCP service

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
