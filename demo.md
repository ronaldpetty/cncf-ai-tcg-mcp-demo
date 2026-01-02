# Demo

This script demonstrates MCP functionality between popular tools including K8sGPT and Claude Desktop.

## k8sgpt MCP Server via cURL

```
# terminal 1
# get OPENAI key
% k8sgpt auth add -b openai -m gpt-4 # bug "max_tokens" needs to become "max_completion_tokens" using 5.z
Enter openai Key: <PASTE_KEY>

% k8sgpt serve --mcp --mcp-http --mcp-port 8089 --backend openai
```

## Tooling

```
# terminal 2
# list prompts
% curl -sX POST http://localhost:8089/mcp \
  -H "Content-Type: application/json" \
  -d '{
    "jsonrpc": "2.0", 
    "id": 1, 
    "method": "tools/list"
  }' | jq '.result.tools[]'
```

```
# terminal 2
# analyze kube-system
% curl -sX POST http://localhost:8089/mcp \
  -H "Content-Type: application/json" \
  -d '{
    "jsonrpc": "2.0",
    "id": 1,
    "method": "tools/call",
    "params": {
      "name": "analyze",
      "arguments": {
        "namespace": "kube-system",
        "explain": true
      }
    }
  }' | jq '.result.content[].text' | xargs printf "%b"
```

## Prompting

```
# terminal 2
# list prompts
% curl -sX POST http://localhost:8089/mcp \
  -H "Content-Type: application/json" \
  -d '{
    "jsonrpc": "2.0", 
    "id": 1, 
    "method": "prompts/list"
  }' | jq '.result.prompts[]'
```

```
# terminal 2
# retrieve trouble shooting prompt
% curl -sX POST http://localhost:8089/mcp \
  -H "Content-Type: application/json" \
  -d '{
    "jsonrpc": "2.0",
    "id": 5,
    "method": "prompts/get",
    "params": {
      "name": "troubleshoot-pod",
      "arguments": {
        "podName": "nginx-abc123",
        "namespace": "default"
      }
    }
  }' | jq '.result.messages[].content.text' | xargs printf "%b"
```

## Resources

```
% curl -sX POST http://localhost:8089/mcp \
  -H "Content-Type: application/json" \
  -d '{
    "jsonrpc": "2.0", 
    "id": 1, 
    "method": "resources/list"
  }' | jq '.result.resources[]'
```

```
% curl -sX POST http://localhost:8089/mcp \
  -H "Content-Type: application/json" \
  -d '{
    "jsonrpc": "2.0",
    "id": 1,
    "method": "resources/read",
    "params": {
        "uri": "cluster-info"
    }
  }' | jq '.result.contents'
```

# k8sgpt MCP Server via Claude Desktop

* stop existing k8sgtp daemon (using port)
* open Claude Desktop (not there is bug and all changes require "force kill" to take affect)

```
# view configuration
% cat "/Users/ronaldpetty/Library/Application Support/Claude/claude_desktop_config.json"
{
  "preferences": {
    "quickEntryDictationShortcut": "capslock"
  },
  "mcpServers": {
    "filesystem": {
      "command": "npx",
      "args": [
        "-y",
        "@modelcontextprotocol/server-filesystem",
        "/Users/ronaldpetty/Desktop",
        "/Users/ronaldpetty/Downloads"
      ]
    },
    "everything": {
      "command": "npx",
      "args": [
        "-y",
        "@modelcontextprotocol/server-everything"
      ]
    },
    "k8sgpt": {
      "command": "k8sgpt",
      "args": ["serve", "--mcp"]
    }
  }
}
```

* demo / talk about "filesystem"
* demo / talk about "everything"
* demo / talk about "k8sgpt"

In Claude Desktop chat, show prompt and rsource connections/apps (e.g. ks8gpt)

## "filesystem" MCP Server STDIO demo

In Claude Desktop chat:

```
# watch logs
% tail -f ~/Library/Logs/Claude/mcp.log
```

* Ask "List available roots"
* Show reasoning
* Show log

* Ask "list files in Desktop"
* Show reasoning
* Show log


## "everything" MCP Server STDIO demo

* Ask "get-sum 1 2"
* Show reasoning
* Show log

* Ask "add 1 2"
* Show reasoning (highlight host did or did not by pass tooling)
* Show log

## k8sgpt MCP Server STDIO demo

* Execute prompts / resources (note it fails in UI but shows in logs)

## Elicitation

* enable sampling (can't in Claude Desktop)

2026-01-01T20:17:44.055Z [info] [everything] Message from server: {"jsonrpc":"2.0","method":"notifications/message","params":{"level":"alert","data":"Alert level-message"}}
2026-01-01T20:17:46.200Z [info] [everything] Message from client: {"method":"tools/call","params":{"name":"sampleLLM","arguments":{"prompt":"What are three benefits of using MCP?"}},"jsonrpc":"2.0","id":7}
2026-01-01T20:17:46.203Z [info] [everything] Message from server: {"jsonrpc":"2.0","id":1,"method":"sampling/createMessage","params":{"messages":[{"role":"user","content":{"type":"text","text":"Resource sampleLLM context: What are three benefits of using MCP?"}}],"systemPrompt":"You are a helpful test server.","maxTokens":100,"temperature":0.7,"includeContext":"thisServer"}}
2026-01-01T20:17:46.210Z [info] [everything] Message from client: {"jsonrpc":"2.0","id":1,"error":{"code":-32601,"message":"Method not found"}}
2026-01-01T20:17:46.211Z [info] [everything] Message from server: {"jsonrpc":"2.0","id":7,"error":{"code":-32601,"message":"MCP error -32601: Method not found"}}