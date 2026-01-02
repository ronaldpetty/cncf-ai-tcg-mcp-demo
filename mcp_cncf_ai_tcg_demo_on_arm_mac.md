# Running K8sGPT on ARM based Mac with Docker Desktop

## Prerequistes

This installs our CRI (aka Docker Desktop).

* Install Docker Desktop
  * https://docs.docker.com/desktop/setup/install/mac-install/

This turns on the include Kubernetes (included in Docker Desktop).

* Install Kubernetes
  * Enter "Settings" for Docker Desktop
  * Click "Kubernetes" on left navigation
  * Click "Enable Kubernetes"
* Confirm Kubernetes is ready
  * Open "Terminal"
  * Run
```
% kubectl get pods -A | grep Running | awk '{print $4}' | sort | uniq -c
   9 Running # depending on version this maybe different, however only "Running" should be returned
```

Install Homebrew to install packages on Mac per https://brew.sh/.

* Install Homebrew (aka `brew`)
  * `/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"`
  * Reload shell (e.g. exit Terminal and reopen)

Install MCP tooling. K8sGPT is a MCP server, while cURL is a start, we leaverage `mcp-tools` as a client per https://github.com/f/mcptools.

* Configure mcptools Homebrew tap
  * `brew tap f/mcptools`
* Install mcptools via Homebrew
  * `brew install mcp`
  * Reload shell (e.g. exit Terminal and reopen)
* Confirm `mcp` is available
```
% mcp version
MCP Tools version 0.7.1
```


## K8sGPT as Binary via Homebrew

This section is based on https://github.com/k8sgpt-ai/k8sgpt?tab=readme-ov-file#cli-installation

* Configure K8sGPT Homebrew tap
  * `brew tap k8sgpt-ai/k8sgpt`
* Install K8sGPT via Homebrew
  * `brew install k8sgpt`
* Confirm K8sGPT is available
  * `k8sgpt analyze` # This is better than "version" as it reaches out to K8s API server


## MCP Server via K8sGPT

This section is based on https://github.com/k8sgpt-ai/k8sgpt/blob/main/MCP.md

* Configure access to OpenAI
  * Create OpenAI access key (e.g. `sk-proj-brg2HVdN0Do...`)
* Configure K8sGPT AI access
```
% k8sgpt auth add -b openai -m gpt-4 # bug "max_tokens" needs to become "max_completion_tokens" using 5.z
Enter openai Key: <PASTE_KEY>
```
* Test AI accessibility from K8sGPT
```
% k8sgpt auth add -b openai -m gpt-4
...
```
* Launch K8sGPT in MCP mode
```
% k8sgpt serve --mcp --mcp-http --mcp-port 8089
{"level":"info","ts":1766857382.6190279,"caller":"server/server.go:149","msg":"binding metrics to 8081"}
{"level":"info","ts":1766857382.6190479,"caller":"server/mcp.go:79","msg":"Starting MCP HTTP server","port":"8089"}
{"level":"info","ts":1766857382.619346,"caller":"server/server.go:108","msg":"binding api to 8080"}
```
* Test MCP server tooling access by listing available MCP tooling
  * Open new terminal tab
```
% curl -sX POST http://localhost:8089/mcp \
  -H "Content-Type: application/json" \
  -d '{
    "jsonrpc": "2.0",
    "id": 1,
    "method": "tools/list"
  }' | jq '.result.tools[] | "\(.name): \(.description)"'
"add-filters: Add filters to enable specific analyzers"
"analyze: Analyze Kubernetes resources for issues and problems"
"cluster-info: Get Kubernetes cluster information and version"
"config: Configure K8sGPT settings including custom analyzers and cache"
"get-logs: Get logs from a pod container"
"get-resource: Get detailed information about a specific Kubernetes resource"
"list-events: List Kubernetes events for debugging and troubleshooting"
"list-filters: List all available and active analyzers/filters in k8sgpt"
"list-integrations: List available integrations (Prometheus, AWS, Keda, Kyverno, etc.)"
"list-namespaces: List all namespaces in the cluster"
"list-resources: List Kubernetes resources of a specific type (pods, deployments, services, nodes, etc.)"
"remove-filters: Remove filters to disable specific analyzers"
```
* Execute MCP Tool `analyze`
```
% curl -sX POST http://localhost:8089/mcp \
  -H "Content-Type: application/json" \
  -d '{
    "jsonrpc": "2.0",
    "id": 1,
    "method": "tools/call",
    "params": {
      "name": "analyze",
      "arguments": {
        "namespace": "kube-system","explain": true
      }
    }
  }' | jq '.result.content[].text' | xargs printf "%b"
AI Provider: openai

0: ConfigMap kube-system/extension-apiserver-authentication()
- Error: ConfigMap extension-apiserver-authentication is not used by any pods in the namespace
Error: The ConfigMap extension-apiserver-authentication is not being used by any pods in the current namespace.
Solution: Ensure that the ConfigMap is correctly linked to the necessary pods. Check the pod configuration files and add the ConfigMap as a volume or an environment variable.
1: ConfigMap kube-system/kube-apiserver-legacy-service-account-token-tracking()

...
```
