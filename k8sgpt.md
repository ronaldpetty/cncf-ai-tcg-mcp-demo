# K8sGPT

Per the website:

> K8sGPT is a tool for scanning your Kubernetes clusters, diagnosing and triaging issues in simple english. It has SRE experience codified into its analyzers and helps to pull out the most relevant information to enrich it with AI.

https://k8sgpt.ai/

You can use Linux or Mac for most of this demo, however the Claude Desktop section requires M-series processors (as of 15-Jan-2026).


## Installation


### Installation on Linux

SSH to VM.

```
% curl -s https://... key.pem
% chmod 700 key.pem
% ssh -i key.pem ubuntu@VM_IP
$
```

```
$ curl -sL https://raw.githubusercontent.com/RX-M/classfiles/refs/heads/master/k8s.sh | sh

# enable command line completion
$ source <(kubectl completion bash)
```

Install K8sGPT.

```
$ sudo dpkg -P k8sgpt
$ curl -sLO https://github.com/k8sgpt-ai/k8sgpt/releases/download/v0.3.48/k8sgpt_amd64.deb
$ sudo dpkg -i k8sgpt_amd64.deb

# enable command line completion
$ source <(k8sgpt completion bash)
```


### Installation on Mac (M-series)

```
# Install Docker Desktop
# Enable Kubernetes
```

```
$ brew tap k8sgpt-ai/k8sgpt
$ brew install k8sgpt
```


## Basic K8sGPT Usage

Without AI, we can get list of identified errors on the Kubernetes cluster:

```
$ k8sgpt analyze

AI Provider: AI not used; --explain not set

No problems detected

$
```

While nothing is found (above), yours output might differ. You may wonder how it works? More on this below, but K8sGPT statically analyses the event stream for know error structures for various resources (aka `k8sgpt filters`).

Deploy something that breaks on the cluster.

```
$ kubectl run brokenpod --image=nginx:unknown_version

pod/brokenpod created

$ k8sgpt analyze

AI Provider: AI not used; --explain not set

0: Pod default/brokenpod()
- Error: Back-off pulling image "nginx:unknown_version"

$
```

Fix it, and confirm its fixed.

```
$ kubectl delete pod brokenpod

pod "brokenpod" deleted

$ kubectl run happypod --image=nginx

pod/happypod created

$ k8sgpt analyze

AI Provider: AI not used; --explain not set

No problems detected

$
```

With AI, we can get a more detailed response:

```
$ k8sgpt analyze --explain
Error: AI provider not specified in configuration. Please run k8sgpt auth
```

View related subcommands.

```
$ k8sgpt auth

Provide the necessary credentials to authenticate with your chosen backend.

Usage:
  k8sgpt auth [flags]
  k8sgpt auth [command]

Available Commands:
  add         Add new provider
  default     Set your default AI backend provider
  list        List configured providers
  remove      Remove provider(s)
  update      Update a backend provider

Flags:
  -h, --help   help for auth

Global Flags:
      --config string        Default config file (/Users/ronaldpetty/Library/Application Support/k8sgpt/k8sgpt.yaml)
      --kubeconfig string    Path to a kubeconfig. Only required if out-of-cluster.
      --kubecontext string   Kubernetes context to use. Only required if out-of-cluster.
  -v, --verbose              Show detailed tool actions (e.g., API calls, checks).

Use "k8sgpt auth [command] --help" for more information about a command.
```

View AI providers, note, local is also an option.

```
$ k8sgpt auth list

Default: 
> openai
Active: 
Unused: 
> openai
> localai
> ollama
> azureopenai
> cohere
> amazonbedrock
> amazonsagemaker
> google
> noopai
> huggingface
> googlevertexai
> oci
> customrest
> ibmwatsonxai
```

There is a helper command to retrieve a key called `generate`.

```
$ k8sgpt generate -h

Opens your browser to generate a key for your chosen backend.

Usage:
  k8sgpt generate [flags]

Flags:
  -b, --backend string   Backend AI provider (default "openai")
  -h, --help             help for generate

Global Flags:
      --config string        Default config file (/Users/ronaldpetty/Library/Application Support/k8sgpt/k8sgpt.yaml)
      --kubeconfig string    Path to a kubeconfig. Only required if out-of-cluster.
      --kubecontext string   Kubernetes context to use. Only required if out-of-cluster.
  -v, --verbose              Show detailed tool actions (e.g., API calls, checks).
```

For now, we use openai (you must have an account). If on Mac, a browser will open. If on Linux server, you will see a message like this.

```
$ k8sgpt generate --backend openai

Please open: https://platform.openai.com/api-keys to generate a key for openai

Please copy the generated key and run `k8sgpt auth add` to add it to your config file

exec: "xdg-open": executable file not found in $PATH

$
```

Create the key and provide read and write abilities via "All" permission.

Next we associate the key with the AI provider (in this case Open AI).

```
$ k8sgpt auth add -h

The add command allows to configure a new backend AI provider

Usage:
  k8sgpt auth add [flags]

Flags:
  -b, --backend string                       Backend AI provider (default "openai")
  -u, --baseurl http://localhost:8080/v1     URL AI provider, (e.g http://localhost:8080/v1)
  -k, --compartmentId string                 Compartment ID for generative AI model (only for oci backend)
  -n, --endpointname endpoint-xxxxxxxxxxxx   Endpoint Name, e.g. endpoint-xxxxxxxxxxxx (only for amazonbedrock, amazonsagemaker backends)
  -e, --engine string                        Azure AI deployment name (only for azureopenai backend)
  -h, --help                                 help for add
  -l, --maxtokens int                        Specify a maximum output length. Adjust (1-...) to control text length. Higher values produce longer output, lower values limit length (default 2048)
  -m, --model string                         Backend AI model (default "gpt-4o")
  -o, --organizationId string                OpenAI or AzureOpenAI Organization ID (only for openai and azureopenai backend)
  -p, --password string                      Backend AI password
  -i, --providerId string                    Provider specific ID for e.g. project (only for googlevertexai/ibmwatsonxai backend)
  -r, --providerRegion string                Provider Region name (only for amazonbedrock, googlevertexai backend)
  -t, --temperature float32                  The sampling temperature, value ranges between 0 ( output be more deterministic) and 1 (more random) (default 0.7)
  -c, --topk int32                           Sampling Cutoff: Set a threshold (1-100) to restrict the sampling process to the top K most probable words at each step. Higher values lead to greater variability, lower values increases predictability. (default 50)
      --topp float32                         Probability Cutoff: Set a threshold (0.0-1.0) to limit word choices. Higher values add randomness, lower values increase predictability. (default 0.5)

Global Flags:
      --config string        Default config file (/Users/ronaldpetty/Library/Application Support/k8sgpt/k8sgpt.yaml)
      --kubeconfig string    Path to a kubeconfig. Only required if out-of-cluster.
      --kubecontext string   Kubernetes context to use. Only required if out-of-cluster.
  -v, --verbose              Show detailed tool actions (e.g., API calls, checks).
```

Paste your token (for Open AI) when asked.

```
$ k8sgpt auth add --backend openai --model gpt-4o-mini

Enter openai Key: openai added to the AI backend provider list

$
```

AI can't solve all problems, yet?!

Before using GPT, lets see how state changes and how it affects K8sGPT (or any tool).

```
$ kubectl delete pod happypod # from earlier

pod "happypod" deleted

$ kubectl run brokenpod --image=nginx:unknown_version

pod/brokenpod created

$
```

Try to analyze a few times.

```
$ k8sgpt analyze

AI Provider: AI not used; --explain not set

0: Pod default/brokenpod()
- Error: Error response from daemon: failed to resolve reference "docker.io/library/nginx:unknown_version": docker.io/library/nginx:unknown_version: not found


$ k8sgpt analyze

AI Provider: AI not used; --explain not set

0: Pod default/brokenpod()
- Error: Back-off pulling image "nginx:unknown_version": ErrImagePull: Error response from daemon: failed to resolve reference "docker.io/library/nginx:unknown_version": docker.io/library/nginx:unknown_version: not found

$
```

Notice how the "state" changes for our broken Pod. This is because, it changes in Kubernetes over time. We can see this happening via "kubectl".

```
# if we ran this in another window
$ kubectl get pods -w

NAME        READY   STATUS    RESTARTS   AGE
brokenpod   0/1     Pending   0          0s
brokenpod   0/1     Pending   0          0s
brokenpod   0/1     ContainerCreating   0          0s
brokenpod   0/1     ErrImagePull        0          2s
brokenpod   0/1     ImagePullBackOff    0          13s
brokenpod   0/1     ErrImagePull        0          24s
brokenpod   0/1     ImagePullBackOff    0          38s
brokenpod   0/1     ErrImagePull        0          52s
```

With state management in mind, we can now see how a GPT (e.g. gpt-4o-mini) can help us via K8sGPT.

```
$ k8sgpt analyze  --explain

 100% |██ ... ██| (1/1, 12 it/min)        
AI Provider: openai

0: Pod default/brokenpod()
- Error: Back-off pulling image "nginx:unknown_version": ErrImagePull: Error response from daemon: failed to resolve reference "docker.io/library/nginx:unknown_version": docker.io/library/nginx:unknown_version: not found
Error: The error indicates that Kubernetes is unable to pull the specified Docker image "nginx:unknown_version" because it cannot find that version in the Docker registry.

Solution: 
1. Check the image name and version for typos.
2. Use a valid version tag (e.g., "nginx:latest").
3. Update your Kubernetes deployment YAML with the correct image.
4. Redeploy the application using `kubectl apply -f your-deployment.yaml`.
```

> n.b. If you get the following error, its most likely your key was incorrectly configured (e.g. "read-only").

```
$ k8sgpt analyze --explain

Error: failed while calling AI provider openai: error, status code: 401, status: 401 Unauthorized, message: Incorrect API key provided: sk-proj-********************************************************************************************************************************************************X0AA. You can find your API key at https://platform.openai.com/account/api-keys.
```

Fix the error.

```
$ kubectl delete pod brokenpod

pod "brokenpod" deleted

$ k8sgpt analyze --explain

AI Provider: openai

No problems detected

$ 
```

What if we didn't know how to fix the error? Lets interact with our GPT to get more information / guidance.

```
$ kubectl run brokenpod --image=nginx:unknown_version

pod/brokenpod created

$ k8sgpt analyze --explain -i

 100% |███ ... ███| (1/1, 5237 it/s)        
AI Provider: openai

0: Pod default/brokenpod()
- Error: Back-off pulling image "nginx:unknown_version": ErrImagePull: Error response from daemon: failed to resolve reference "docker.io/library/nginx:unknown_version": docker.io/library/nginx:unknown_version: not found
Error: The error indicates that Kubernetes is unable to pull the specified Docker image "nginx:unknown_version" because it cannot find that version in the Docker registry.

Solution: 
1. Check the image name and version for typos.
2. Use a valid version tag (e.g., "nginx:latest").
3. Update your Kubernetes deployment YAML with the correct image.
4. Redeploy the application using `kubectl apply -f your-deployment.yaml`.

Interactive mode enabled [type exit to close.]
Input text: 
```

Lets ask for help ("Can you tell me the kubectl command to run to fix this issue?").

```
Input text: Can you tell me the kubectl command to run to fix this issue?

To address the issues mentioned in the context, you would need to run different `kubectl` commands based on the specific problem you're trying to fix. Here are the commands for each issue:

1. **For the Pod error regarding the image "nginx:unknown_version":**
   - First, you need to update your deployment YAML file to use a valid image tag (e.g., "nginx:latest"). After updating the YAML file, you can redeploy the application with the following command:
     ```bash
     kubectl apply -f your-deployment.yaml

Make sure to replace `your-deployment.yaml` and `<namespace>` with the actual names relevant to your environment.
Input text:     
```

Thanks AI! While we didn't use a deployment, if you know K8s, its not wrong. Exit and cleanup.

```
Input text: ^C

$ kubectl delete pod brokenpod

pod "brokenpod" deleted
```

Lets check another condition, where we deploy a misconfigured service.

```
$ kubectl create service clusterip --tcp=80:80 to-nowhere

service/to-nowhere created

$ k8sgpt analyze

AI Provider: AI not used; --explain not set

0: Service default/to-nowhere()
- Error: Service has no endpoints, expected label app=to-nowhere

# try with JSON output formatting
$ k8sgpt analyze --explain -o json

{
  "provider": "openai",
  "errors": null,
  "status": "ProblemDetected",
  "problems": 1,
  "results": [
    {
      "kind": "Service",
      "name": "default/to-nowhere",
      "error": [
        {
          "Text": "Service has no endpoints, expected label app=to-nowhere",
          "KubernetesDoc": "",
          "Sensitive": [
            {
              "Unmasked": "app",
              "Masked": "d1gi"
            },
            {
              "Unmasked": "to-nowhere",
              "Masked": "ZSFmIjNUZjlyMg=="
            }
          ]
        }
      ],
      "details": "Error: The service cannot route traffic because it has no endpoints, meaning there are no pods matching the specified label `app=to-nowhere`.\n\nSolution: \n1. Check if the deployment for `app=to-nowhere` exists.\n2. Ensure pods are running: `kubectl get pods -l app=to-nowhere`.\n3. Verify the label on the pods: `kubectl describe pod \u003cpod-name\u003e`.\n4. If missing, update the deployment or service with the correct labels.",
      "parentObject": ""
    }
  ]
}
```

We can confirm this via `kubectl`.

```
$ kubectl get svc,endpoints

NAME                                          TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE
service/kubernetes                            ClusterIP   10.96.0.1        <none>        443/TCP   128d
service/to-nowhere                            ClusterIP   10.107.91.217    <none>        80/TCP    71s

NAME                                            ENDPOINTS           AGE
endpoints/kubernetes                            192.168.65.3:6443   128d
endpoints/to-nowhere                            <none>              71s

$ kubectl get svc to-nowhere -o wide

NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE    SELECTOR
to-nowhere   ClusterIP   10.107.91.217   <none>        80/TCP    114s   app=to-nowhere

$ kubectl run nginx --image=nginx -l app=to-nowhere

pod/nginx created

$ kubectl get svc,endpoints

NAME                                          TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE
service/kubernetes                            ClusterIP   10.96.0.1        <none>        443/TCP   128d
service/to-nowhere                            ClusterIP   10.107.91.217    <none>        80/TCP    2m11s

NAME                                            ENDPOINTS           AGE
endpoints/kubernetes                            192.168.65.3:6443   128d
endpoints/to-nowhere                            10.1.1.53:80        2m11s
```

Now that we have a destination, the journey should be fixed.

```
$ k8sgpt analyze

AI Provider: AI not used; --explain not set

No problems detected

$ k8sgpt analyze --explain

AI Provider: openai

No problems detected

$
```

Congratulations, you have used AI to assist you debug and fix Kubernetes deployment issues.


## Configuration

K8sGPT can be configured via the command line interface or via direct editing of the related configuration file. Depending on which method you used to install K8sGPT, it may be in different location (try `k8sgpt | grep k8sgpt.yaml` to see location).

```
$ cat .config/k8sgpt/k8sgpt.yaml

ai:
    providers:
        - name: openai
          model: gpt-4o-mini
          password: sk-proj-...
          temperature: 0.7
          topp: 0.5
          topk: 50
          maxtokens: 2048
          customheaders: []
    defaultprovider: ""
commit: Homebrew
date: "2025-12-18T13:52:08Z"
kubeconfig: ""
kubecontext: ""
verbose: false
version: 0.4.27

$
```

Most commands that configure or use the backend supply `-b / --backend`. Some commands such as `-e / --explain` use the default backend (if configured). Unfortunately, the terms "backend" and "provider" overlap.

We won't cover the backed provider configuration details here (those are specific to the provider). To learn about Open AI Platform API options see:

https://platform.openai.com/docs/api-reference/introduction

Now we look at how K8sGPT views K8s resources. The term "filters" refers to Resources and Customer Resources in K8s. These objects are what K8sGPT looks for when running `analyze`.

```
$ k8sgpt filters list

Active: 
> Job
> Node
> Pod
> Deployment
> ReplicaSet
> Service
> CronJob
> ValidatingWebhookConfiguration
> MutatingWebhookConfiguration
> ConfigMap
> PersistentVolumeClaim
> Ingress
> StatefulSet
Unused: 
> HorizontalPodAutoscaler
> PodDisruptionBudget
> GatewayClass
> Storage
> ClusterCatalog
> ClusterExtension
> Subscription
> CatalogSource
> NetworkPolicy
> Log
> Gateway
> HTTPRoute
> Security
> OperatorGroup
> ClusterServiceVersion
> InstallPlan
```

We can see Pod and Service K8s resources listed under a filter. 

We can deactivate an active filter so it is not considered during review (via `analyze`). Lets deactivate Pod and see a bad pod is not in our results.

```
$ kubectl delete pod brokenpod nginx
$ kubectl delete svc to-nowhere
$ kubectl run brokenpod --image=nginx:i_dont_exist

$ k8sgpt analyze

AI Provider: AI not used; --explain not set

0: Pod default/brokenpod()
- Error: Back-off pulling image "nginx:i_dont_exist"

$ k8sgpt filters remove Pod

Filter(s) Pod removed

~$ k8sgpt filters list
Active: 
> ValidatingWebhookConfiguration
> MutatingWebhookConfiguration
> ConfigMap
> Deployment
> ReplicaSet
> Service
> Ingress
> Job
> CronJob
> Node
> PersistentVolumeClaim
> StatefulSet
Unused: 
> Pod
> HTTPRoute
> PodDisruptionBudget
> Gateway
> Security
> ClusterCatalog
> ClusterExtension
> CatalogSource
> Storage
> ClusterServiceVersion
> Subscription
> InstallPlan
> NetworkPolicy
> OperatorGroup
> HorizontalPodAutoscaler
> Log
> GatewayClass

~$ k8sgpt analyze

AI Provider: AI not used; --explain not set

No problems detected

~$ kubectl get pods

NAME        READY   STATUS             RESTARTS   AGE
brokenpod   0/1     ImagePullBackOff   0          92s

~$ k8sgpt filters add Pod

Filter Pod added

~$ k8sgpt analyze

AI Provider: AI not used; --explain not set

0: Pod default/brokenpod()
- Error: rpc error: code = NotFound desc = failed to pull and unpack image "docker.io/library/nginx:i_dont_exist": failed to resolve reference "docker.io/library/nginx:i_dont_exist": docker.io/library/nginx:i_dont_exist: not found
```


## Customizing

While K8sGPT is open source, and we could continue to modify its code base, it can be challenging. To help minimize impact, we can use K8sGPT Custom Analyzers (CA). A CA is a service that can integrate with K8sGPT via the network. In the standard form, its used to probe new systems and report issues back to K8sGPT where analysis can happen. K8sGPT queries these agents and runs GPT style queries. Think of this as a precursor to MCP (which we discuss later).

Like other resources, we can list, add, remove CAs.

```
$ k8sgpt custom-analyzer list

Active:
```

Lets deploy the hello world of CAs by making it!

```
$ mkdir ~/my_ca && cd ~/my_ca

# our gRPC server
~/my_ca$ vi main.go 

package main

import (
	"errors"
	"fmt"
	"net"
	"net/http"

	rpc "buf.build/gen/go/k8sgpt-ai/k8sgpt/grpc/go/schema/v1/schemav1grpc"
	//	"github.com/k8sgpt-ai/go-custom-analyzer/pkg/analyzer"
	"google.golang.org/grpc"
	"google.golang.org/grpc/reflection"
)

func main() {
	var err error
	address := fmt.Sprintf(":%s", "8085")
	lis, err := net.Listen("tcp", address)
	if err != nil {
		panic(err)
	}
	grpcServer := grpc.NewServer()
	reflection.Register(grpcServer)
	aa := Analyzer{}
	rpc.RegisterCustomAnalyzerServiceServer(grpcServer, aa.Handler)
	fmt.Println("Starting!!")
	if err := grpcServer.Serve(
		lis,
	); err != nil && !errors.Is(err, http.ErrServerClosed) {
		return
	}
}

# our actual analysis provider (loaded by gRPC above)
~/my_ca$ vi analyzer.go

package main

import (
	"context"
	"fmt"

	rpc "buf.build/gen/go/k8sgpt-ai/k8sgpt/grpc/go/schema/v1/schemav1grpc"
	v1 "buf.build/gen/go/k8sgpt-ai/k8sgpt/protocolbuffers/go/schema/v1"
	"github.com/ricochet2200/go-disk-usage/du"
)

type Handler struct {
	rpc.CustomAnalyzerServiceServer
}
type Analyzer struct {
	Handler *Handler
}

func (a *Handler) Run(context.Context, *v1.RunRequest) (*v1.RunResponse, error) {
	println("Running analyzer")
	usage := du.NewDiskUsage("/")
	diskUsage := int((usage.Size() - usage.Free()) * 100 / usage.Size())
	return &v1.RunResponse{
		Result: &v1.Result{
			Name:    "diskuse",
			Details: fmt.Sprintf("Disk usage is %d", diskUsage),
			Error: []*v1.ErrorDetail{
				{
					Text: fmt.Sprintf("Disk usage is %d", diskUsage),
				},
			},
		},
	}, nil
}
```

Next we compile and launch.

```
~/my_ca$ go mod init test

~/my_ca$ go mod tidy

~/my_ca$ go run main.go analyzer.go
```

Next we configure K8sGPT so it can reach this new CA service.

```
# in another terminal
$ k8sgpt custom-analyzer add -n my-ca -r 8085 -u localhost

my-ca added to the custom analyzers config list

$ k8sgpt custom-analyzer list --details

Active: 
> my-ca
   - Url: localhost
   - Port: 8085
```

The prior add subcommand injected the information into our configuration file.

```
$ cat .config/k8sgpt/k8sgpt.yaml

active_filters:
    - ValidatingWebhookConfiguration
    - MutatingWebhookConfiguration
    - ConfigMap
    - Deployment
    - Job
    - PersistentVolumeClaim
    - StatefulSet
    - ReplicaSet
    - Service
    - Ingress
    - CronJob
    - Node
    - Pod
ai:
    defaultprovider: ""
    providers:
        - customheaders: []
          maxtokens: 2048
          model: gpt-4o-mini
          name: openai
          password: sk-proj-...
          temperature: 0.7
          topk: 50
          topp: 0.5
commit: Homebrew
custom_analyzers:
    - name: my-ca
      connection:
        url: localhost
        port: 8085
date: "2025-12-18T13:52:08Z"
kubeconfig: ""
kubecontext: ""
verbose: false
version: 0.4.27
```

To activate this service, we use the --custom-analysis option.

```
~$ k8sgpt analyze --custom-analysis

AI Provider: AI not used; --explain not set

0: my-ca diskuse()
- Error: Disk usage is 6
Disk usage is 6
1: Pod default/brokenpod()
- Error: Back-off pulling image "nginx:i_dont_exist"

~$ k8sgpt analyze --custom-analysis --explain

 100% |██ ... ███| (2/2, 10000 it/s)        
AI Provider: openai

0: my-ca diskuse()
- Error: Disk usage is 6
Error: The disk usage is at 6, indicating that the storage may be nearing its limit or has reached a threshold that could affect performance.

Solution: 
1. Check disk usage with `df -h`.
2. Identify large files with `du -sh *`.
3. Delete unnecessary files or logs.
4. Consider increasing disk size if needed.
5. Monitor usage regularly to prevent future issues.
1: Pod default/brokenpod()
- Error: Back-off pulling image "nginx:i_dont_exist"
Error: The error indicates that Kubernetes is unable to pull the specified Docker image "nginx:i_dont_exist" because it does not exist in the repository.

Solution: 
1. Check the image name for typos.
2. Verify the image tag exists in the repository.
3. If using a private repository, ensure proper authentication is set up.
4. Update the deployment with the correct image name/tag.

$
```

You can learn more here https://docs.k8sgpt.ai/tutorials/custom-analyzers/.


## MCP


### Codex (Mac instructions - Intel processor but should work on M-Series as well)

```
$ brew install codex

$ mkdir ./codex_k8sgpt_via_mcp && cd ./codex_k8sgpt_via_mcp
```

```
$ export OPENAI_API_KEY=sk-proj-...

$ codex login status

Not logged in

$ printenv OPENAI_API_KEY | codex login --with-api-key

Reading API key from stdin...
Successfully logged in

$ codex login status                                  

Logged in using an API key - sk-proj-***yt5EA
```

Now we integrate our MCP configuration (K8sGPT via STDIO).

```
$ codex_k8sgpt_via_mcp % codex mcp list

No MCP servers configured yet. Try `codex mcp add my-tool -- my-command`.

$ codex mcp add k8sgpt -- k8sgpt serve --mcp

Added global MCP server 'k8sgpt'.

$ codex mcp list

Name    Command  Args         Env  Cwd  Status   Auth       
k8sgpt  k8sgpt   serve --mcp  -    -    enabled  Unsupported
```

Now we can interact with K8sGPT via Codex.

```
$ codex mcp get k8sgpt

k8sgpt
  enabled: true
  transport: stdio
  command: k8sgpt
  args: serve --mcp
  cwd: -
  env: -
  remove: codex mcp remove k8sgpt

$ 
```

```
$ codex
```

```
> You are running Codex in /Users/ronaldpetty/codex_k8sgpt_via_mcp
 
  Since this folder is not version controlled, we recommend requiring approval of all edits and commands.
 
  1. Allow Codex to work in this folder without asking for approval
› 2. Require approval of edits and commands
 
  Press enter to continue
```

* Press "enter"

```
╭─────────────────────────────────────────────────╮
│ ✨ Update available! 0.84.0 -> 0.85.0           │
│ Run brew upgrade codex to update.               │
│                                                 │
│ See full release notes:                         │
│ https://github.com/openai/codex/releases/latest │
╰─────────────────────────────────────────────────╯

╭─────────────────────────────────────────────╮
│ >_ OpenAI Codex (v0.84.0)                   │
│                                             │
│ model:     gpt-5.2-codex   /model to change │
│ directory: ~/codex_k8sgpt_via_mcp           │
╰─────────────────────────────────────────────╯

  To get started, describe a task or try one of these commands:

  /init - create an AGENTS.md file with instructions for Codex
  /status - show current session configuration
  /approvals - choose what Codex can do without approval
  /model - choose what model and reasoning effort to use
  /review - review any changes and find issues

 
› Summarize recent commits
 
  100% context left · ? for shortcuts
```

List MCP services.

```
/mcp

🔌  MCP Tools

  • k8sgpt
    • Status: enabled
    • Auth: Unsupported
    • Command: k8sgpt serve --mcp
    • Tools: add-filters, analyze, cluster-info, config, get-logs, get-resource, list-events, list-filters, list-integrations, list-namespaces, list-resources, remove-filters
    • Resources: active-filters (active-filters), cluster-info (cluster-info), namespaces (namespaces)
    • Resource templates: (none)


 
› Summarize recent commits
 
  100% context left · ? for shortcuts
```

We see the available tools and resources. Lets call "list-namespaces".

```
› #list-namespaces


• Called k8sgpt.list-namespaces({})
  └ [{"metadata": {"name": "default", "uid": "899ec73e-29c1-4f74-82e1-549bd39ffbe5", "resourceVersion": "24", "creationTimestamp": "2025-09-09T02:53:31Z", "labels": {"kubernetes.io/metadata.name": "default"}, "managedFields": [{"manager": "kube-
        apiserver", "operation": "Update", "apiVersion": "v1", "time": "2025-09-09T02:53:31Z", "fieldsType": "FieldsV1", "fieldsV1": {"f:metadata": {"f:labels": {".": {}, "f:kubernetes.io/metadata.name": {}}}}}]}, "spec": {"finalizers":
        ["kubernetes"]}, "status": {"phase": "Active"}}, {"metadata": {"name": "envoy-ai-gateway-system", "uid": "67a7a236-794b-40f7-9cba-739e3cfbe422", "resourceVersion": "295434", "creationTimestamp": "2026-01-14T19:13:54Z", "labels":
        {"kubernetes.io/metadata.name": "envoy-ai-gateway-system", "name": "envoy-ai-gateway-system"}, "managedFields": [{"manager": "helm", "operation": "Apply", "apiVersion": "v1", "time": "2026-01-14T19:13:54Z", "fieldsType": "FieldsV1",
        "fieldsV1": {"f:metadata": {"f:labels": {"f:name": {}}}, "f:spec": {}}}]}, "spec": {"finalizers": ["kubernetes"]}, "status": {"phase": "Active"}}, {"metadata": {"name": "envoy-gateway-system", "uid": "a2844ee2-a46c-45d6-816b-62218d44571a",
        "resourceVersion": "303729", "creationTimestamp": ...

─ Worked for 3s ──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────

• Namespaces:

  - default
  - envoy-ai-gateway-system
  - envoy-gateway-system
  - gateway-system
  - kube-node-lease
  - kube-public
  - kube-system
```

Now lets "analyze"

```
› #analyze


• Called k8sgpt.analyze({})
  └ {"provider": "", "errors": null, "status": "ProblemDetected", "problems": 16, "results": [{"kind": "Deployment", "name": "envoy-gateway-system/envoy-gateway", "error": [{"Text": "Deployment envoy-gateway-system/envoy-gateway has 1 replicas but 0
        are available with status running", "KubernetesDoc": "", "Sensitive": [{"Unmasked": "envoy-gateway-system", "Masked": "ZCRtSSUlQUd0aTczZ00tZiFQeiE="}, {"Unmasked": "envoy-gateway", "Masked": "Q1ExbHxyVHpCU3AkNw=="}]}], "details": "",
        "parentObject": ""}, {"kind": "ConfigMap", "name": "default/kube-root-ca.crt", "error": [{"Text": "ConfigMap kube-root-ca.crt is not used by any pods in the namespace", "KubernetesDoc": "", "Sensitive": []}], "details": "", "parentObject":
        ""}, {"kind": "ConfigMap", "name": "envoy-ai-gateway-system/kube-root-ca.crt", "error": [{"Text": "ConfigMap kube-root-ca.crt is not used by any pods in the namespace", "KubernetesDoc": "", "Sensitive": []}], "details": "", "parentObject":
        ""}, {"kind": "ConfigMap", "name": "envoy-gateway-system/kube-root-ca.crt", "error": [{"Text": "ConfigMap kube-root-ca.crt is not used by any pods in the namespace", "KubernetesDoc": "", "Sensitive": []}], "details": "", "parentObject": ""},
        {"kind": "ConfigMap"...

─ Worked for 5s ──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────

• Analysis results:

  - Deployment envoy-gateway-system/envoy-gateway has 1 replica but 0 available.
  - Pod default/brokenpod is in ErrImagePull for nginx:i_dont_exist.
  - Services envoy-gateway-system/envoy-gateway and envoy-gateway-system/envoy-gateway-metrics-service have no ready endpoints.
  - Multiple ConfigMaps reported as unused (kube-root-ca.crt, cluster-info, etc.). These are often benign system defaults.

  Want me to drill into the failing deployment/pod (events/logs) or filter the analysis to a specific namespace?

 
› Summarize recent commits
 
  100% context left · ? for shortcuts
```

> TODO - find fix in K8sGPT / Codex MCP prompt management (appears missing in Codex)


### Claude Desktop (M-Series required)

On M-Series Mac.

* Create Anthropic account
* Install Claude Desktop
  * Your access to Claude is via Desktop (so no "key", just login)
  * K8sGPT is using OpenAI per its configuration earlier

```
# edit configuration
$ vi "/Users/ronaldpetty/Library/Application Support/Claude/claude_desktop_config.json"
{
  "preferences": {
    "quickEntryDictationShortcut": "capslock"
  },
  "mcpServers": {
    "k8sgpt": {
      "command": "k8sgpt",
      "args": ["serve", "--mcp"]
    }
  }
}
```

* Launch Claude (else stop and launch again as configuration is start time only)

In Claude Desktop chat, show prompt and rsource connections/apps (e.g. ks8gpt).


## Next Steps

Provide feedback or code to K8sGPT!

Documentation:

* https://docs.k8sgpt.ai/

Slack

* k8sgpt.slack.com