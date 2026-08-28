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
$ export DOCKER_VERSION=29.6.2 && curl -sL https://raw.githubusercontent.com/RX-M/classfiles/refs/heads/master/k8s.sh | sh

# enable command line completion
$ source <(kubectl completion bash)
```

Install K8sGPT.

```
$ sudo dpkg -P k8sgpt
$ curl -sLO https://github.com/k8sgpt-ai/k8sgpt/releases/download/v0.4.36/k8sgpt_amd64.deb
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
$ k8sgpt analyze -n default

AI Provider: AI not used; --explain not set

No problems detected

$
```

While nothing is found (above), yours output might differ. You may wonder how it works? More on this below, but K8sGPT statically analyses the event stream for know error structures for various resources (aka `k8sgpt filters`).

Deploy something that breaks on the cluster.

```
$ kubectl run brokenpod --image=nginx:unknown_version

pod/brokenpod created

$ k8sgpt analyze -n default

AI Provider: AI not used; --explain not set

0: Pod default/brokenpod()
- Error: Back-off pulling image "nginx:unknown_version": ErrImagePull: rpc error: code = NotFound desc = failed to pull and unpack image "docker.io/library/nginx:unknown_version": failed to resolve image: docker.io/library/nginx:unknown_version: not found

$
```

Fix it, and confirm its fixed.

```
$ kubectl delete pod brokenpod

pod "brokenpod" deleted

$ kubectl run happypod --image=nginx

pod/happypod created

$ k8sgpt analyze -n default

AI Provider: AI not used; --explain not set

No problems detected

$
```

With AI, we can get a more detailed response:

```
$ k8sgpt analyze -n default --explain
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
      --config string        Default config file (/home/ubuntu/.config/k8sgpt/k8sgpt.yaml)
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
> anthropic
> localai
> ollama
> azureopenai
> cohere
> amazonbedrock
> amazonbedrockconverse
> amazonsagemaker
> google
> noopai
> huggingface
> googlevertexai
> oci
> customrest
> ibmwatsonxai
> groq
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
      --config string        Default config file (/home/ubuntu/.config/k8sgpt/k8sgpt.yaml)
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
  -a, --azureAPIType string                  AzureOpenAI API Type name. Valid values: AZURE, AZURE_AD or CLOUDFLARE_AZURE (only for azureopenai backend)
      --azureAPIVersion string               AzureOpenAI API version, e.g. 2024-02-15-preview (only for azureopenai backend)
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
  -r, --providerRegion string                Provider Region name (only for amazonbedrock, amazonbedrockconverse, googlevertexai backend)
  -s, --stopsequences strings                Stop Sequences: Define specific tokens or phrases that signal the model to stop generating text.
  -t, --temperature float32                  The sampling temperature, value ranges between 0 ( output be more deterministic) and 1 (more random) (default 0.7)
  -c, --topk int32                           Sampling Cutoff: Set a threshold (1-100) to restrict the sampling process to the top K most probable words at each step. Higher values lead to greater variability, lower values increases predictability. (default 50)
      --topp float32                         Probability Cutoff: Set a threshold (0.0-1.0) to limit word choices. Higher values add randomness, lower values increase predictability. (default 0.5)

Global Flags:
      --config string        Default config file (/home/ubuntu/.config/k8sgpt/k8sgpt.yaml)
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
$ k8sgpt analyze -n default

AI Provider: AI not used; --explain not set

0: Pod default/brokenpod()
- Error: rpc error: code = NotFound desc = failed to pull and unpack image "docker.io/library/nginx:unknown_version": failed to resolve image: docker.io/library/nginx:unknown_version: not found

$ k8sgpt analyze -n default

AI Provider: AI not used; --explain not set

0: Pod default/brokenpod()
- Error: Back-off pulling image "nginx:unknown_version": ErrImagePull: rpc error: code = NotFound desc = failed to pull and unpack image "docker.io/library/nginx:unknown_version": failed to resolve image: docker.io/library/nginx:unknown_version: not found

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
$ k8sgpt analyze -n default --explain

 100% |██ ... ██| (1/1, 12 it/min)        
AI Provider: openai

0: Pod default/brokenpod()
- Error: rpc error: code = NotFound desc = failed to pull and unpack image "docker.io/library/nginx:unknown_version": failed to resolve image: docker.io/library/nginx:unknown_version: not found
Error: The error indicates that Kubernetes cannot find the specified Docker image "nginx:unknown_version" on Docker Hub.

Solution:
1. Check the image name and tag for typos.
2. Verify the image exists on Docker Hub.
3. Use a valid tag (e.g., "nginx:latest").
4. Update your Kubernetes deployment configuration with the correct image.
5. Redeploy the application.
```

> n.b. If you get the following error, its most likely your key was incorrectly configured (e.g. "read-only").

```
$ k8sgpt analyze -n default --explain

Error: failed while calling AI provider openai: error, status code: 401, status: 401 Unauthorized, message: Incorrect API key provided: sk-proj-********************************************************************************************************************************************************X0AA. You can find your API key at https://platform.openai.com/account/api-keys.
```

Fix the error.

```
$ kubectl delete pod brokenpod

pod "brokenpod" deleted

$ k8sgpt analyze -n default --explain

AI Provider: openai

No problems detected

$ 
```

What if we didn't know how to fix the error? Lets interact with our GPT to get more information / guidance.

```
$ kubectl run brokenpod --image=nginx:unknown_version

pod/brokenpod created

$ k8sgpt analyze -n default --explain -i

 100% |███ ... ███| (1/1, 5237 it/s)        
AI Provider: openai

0: Pod default/brokenpod()
- Error: rpc error: code = NotFound desc = failed to pull and unpack image "docker.io/library/nginx:unknown_version": failed to resolve image: docker.io/library/nginx:unknown_version: not found
Error: The error indicates that Kubernetes cannot find the specified Docker image "nginx:unknown_version" on Docker Hub.

Solution:
1. Check the image name and tag for typos.
2. Verify the image exists on Docker Hub.
3. Use a valid tag (e.g., "nginx:latest").
4. Update your Kubernetes deployment configuration with the correct image.
5. Redeploy the application.

Interactive mode enabled [type exit to close.]
Input text: 
```

Lets ask for help ("Can you tell me the kubectl command to run to fix this issue?").

````
Input text: Can you tell me the kubectl command to run to fix this issue?

To address the issues mentioned in the context, you will need to run different `kubectl` commands based on the specific problems.

### For the issue (Broken Pod due to image not found):

1. **Check the current deployment configuration**:
   ```bash
   kubectl get deployments -n default
   ```

2. **Edit the deployment to correct the image tag** (assuming the deployment name is `your-deployment-name`):
   ```bash
   kubectl edit deployment your-deployment-name -n default
   ```
   In the editor, locate the image line and change `nginx:unknown_version` to a valid version, such as `nginx:latest`.

3. **Redeploy the application** (if you updated the deployment):
   ```bash
   kubectl rollout restart deployment your-deployment-name -n default


### Summary of Commands:
- To edit a deployment:
  ```bash
  kubectl edit deployment your-deployment-name -n default
  ```
- To restart a deployment:
  ```bash
  kubectl rollout restart deployment your-deployment-name -n default
  ```

Make sure to replace `your-deployment-name` with the actual name of your deployment.   

Make sure to replace `your-deployment.yaml` and `<namespace>` with the actual names relevant to your environment.
Input text:     
````

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

$ k8sgpt analyze -n default

AI Provider: AI not used; --explain not set

0: Service default/to-nowhere()
- Error: Service has no endpoints, expected label app=to-nowhere

# try with JSON output formatting
$ k8sgpt analyze -n default --explain -o json

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
              "Masked": "cio/"
            },
            {
              "Unmasked": "to-nowhere",
              "Masked": "ekMzPnlWaVNUVw=="
            }
          ]
        }
      ],
      "details": "Error: The service is not routing traffic because it has no endpoints, meaning there are no pods matching the specified label `app=to-nowhere`.\n\nSolution: \n1. Check if any pods are running: `kubectl get pods`\n2. Verify pod labels: `kubectl get pods --show-labels`\n3. Ensure pods have the label `app=to-nowhere`.\n4. If not, update pod labels or modify the service to match existing labels.",
      "parentObject": ""
    }
  ]
}
```

We can confirm this via `kubectl`.

```
$ kubectl get svc,endpoints

NAME                 TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
service/kubernetes   ClusterIP   10.96.0.1       <none>        443/TCP   16m
service/to-nowhere   ClusterIP   10.109.13.112   <none>        80/TCP    65s

NAME                   ENDPOINTS           AGE
endpoints/kubernetes   172.31.18.14:6443   16m
endpoints/to-nowhere   <none>              65s

$ kubectl get svc to-nowhere -o wide

NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE   SELECTOR
to-nowhere   ClusterIP   10.109.13.112   <none>        80/TCP    83s   app=to-nowhere

$ kubectl run nginx --image=nginx -l app=to-nowhere

pod/nginx created

$ kubectl get svc,endpoints

NAME                 TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
service/kubernetes   ClusterIP   10.96.0.1       <none>        443/TCP   16m
service/to-nowhere   ClusterIP   10.109.13.112   <none>        80/TCP    101s

NAME                   ENDPOINTS           AGE
endpoints/kubernetes   172.31.18.14:6443   16m
endpoints/to-nowhere   10.0.0.142:80       101s
```

Now that we have a destination, the journey should be fixed.

```
$ k8sgpt analyze -n default

AI Provider: AI not used; --explain not set

No problems detected

$ k8sgpt analyze -n default --explain

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
          password: sk-proj-K...
          temperature: 0.7
          topp: 0.5
          topk: 50
          maxtokens: 2048
          customheaders: []
    defaultprovider: ""
commit: d33935a
date: unknown
kubeconfig: ""
kubecontext: ""
verbose: false
version: 0.4.36

$
```

You can find the default location of the configuration file by looking at help menu.

```
% k8sgpt -h | grep Default
      --config string        Default config file (/home/ubuntu/.config/k8sgpt/k8sgpt.yaml)
```

Most commands that configure or use the backend supply `-b / --backend`. Some commands such as `-e / --explain` use the default backend (if configured). Unfortunately, the terms "backend" and "provider" overlap.

We won't cover the backed provider configuration details here (those are specific to the provider). To learn about Open AI Platform API options see:

https://platform.openai.com/docs/api-reference/introduction

Now we look at how K8sGPT views K8s resources. The term "filters" refers to Resources and Customer Resources in K8s. These objects are what K8sGPT looks for when running `analyze`.

```
$ k8sgpt filters list

Active:
> Service
> ValidatingWebhookConfiguration
> MutatingWebhookConfiguration
> ConfigMap
> DaemonSet
> PersistentVolumeClaim
> Ingress
> StatefulSet
> Job
> CronJob
> Node
> Pod
> Deployment
> ReplicaSet
Unused:
> Gateway
> Subscription
> OperatorGroup
> HorizontalPodAutoscaler
> HTTPRoute
> InstallPlan
> NetworkPolicy
> GatewayClass
> ClusterCatalog
> PodDisruptionBudget
> Log
> Storage
> Security
> ClusterExtension
> ClusterServiceVersion
> CatalogSource
```

We can see Pod and Service K8s resources listed under a filter. 

We can deactivate an active filter so it is not considered during review (via `analyze`). Lets deactivate Pod and see a bad pod is not in our results.

```
$ kubectl delete pod brokenpod nginx
$ kubectl delete svc to-nowhere
$ kubectl run brokenpod --image=nginx:i_dont_exist

$ k8sgpt analyze -n default

AI Provider: AI not used; --explain not set

0: Pod default/brokenpod()
- Error: rpc error: code = NotFound desc = failed to pull and unpack image "docker.io/library/nginx:i_dont_exist": failed to resolve image: docker.io/library/nginx:i_dont_exist: not found

$ k8sgpt filters remove Pod

Filter(s) Pod removed

~$ k8sgpt filters list

Active:
> PersistentVolumeClaim
> Service
> ValidatingWebhookConfiguration
> MutatingWebhookConfiguration
> ConfigMap
> DaemonSet
> ReplicaSet
> Ingress
> StatefulSet
> Job
> CronJob
> Node
> Deployment
Unused:
> Pod
> Log
> PodDisruptionBudget
> GatewayClass
> HTTPRoute
> Storage
> Security
> ClusterCatalog
> ClusterExtension
> ClusterServiceVersion
> Subscription
> InstallPlan
> OperatorGroup
> HorizontalPodAutoscaler
> Gateway
> CatalogSource
> NetworkPolicy

$ k8sgpt analyze -n default

AI Provider: AI not used; --explain not set

No problems detected

$ kubectl get pods

NAME        READY   STATUS             RESTARTS   AGE
brokenpod   0/1     ImagePullBackOff   0          92s

$ k8sgpt filters add Pod

Filter Pod added

$ k8sgpt analyze -n default

AI Provider: AI not used; --explain not set

0: Pod default/brokenpod()
- Error: rpc error: code = NotFound desc = failed to pull and unpack image "docker.io/library/nginx:i_dont_exist": failed to resolve image: docker.io/library/nginx:i_dont_exist: not found
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
~/my_ca$ sudo apt install golang-go -y

~/my_ca$ go mod init test

...

~/my_ca$ go mod tidy

...

~/my_ca$ CGO_ENABLED=0 go run main.go analyzer.go

Starting!!
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

Active:
> my-ca
   - Url: localhost
   - Port: 8085
ubuntu@ip-172-31-18-14:~$ cat .config/k8sgpt/k8sgpt.yaml
active_filters:
    - Ingress
    - CronJob
    - Deployment
    - Pod
    - PersistentVolumeClaim
    - ValidatingWebhookConfiguration
    - MutatingWebhookConfiguration
    - ConfigMap
    - DaemonSet
    - StatefulSet
    - Job
    - Node
    - Service
    - ReplicaSet
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
commit: d33935a
custom_analyzers:
    - name: my-ca
      connection:
        url: localhost
        port: 8085
date: unknown
kubeconfig: ""
kubecontext: ""
verbose: false
version: 0.4.36
```

To activate this service, we use the --custom-analysis option.

```
$ k8sgpt analyze -n default --custom-analysis

AI Provider: AI not used; --explain not set

0: my-ca diskuse()
- Error: Disk usage is 3
Disk usage is 3
1: Pod default/brokenpod()
- Error: Back-off pulling image "nginx:i_dont_exist": ErrImagePull: rpc error: code = NotFound desc = failed to pull and unpack image "docker.io/library/nginx:i_dont_exist": failed to resolve image: docker.io/library/nginx:i_dont_exist: not found

~$ k8sgpt analyze -n default --custom-analysis --explain

 100% |██ ... ███| (2/2, 10000 it/s)        
AI Provider: openai

0: my-ca diskuse()
- Error: Disk usage is 3
Error: The disk usage is at 3, indicating that the storage space is nearly full or has reached a critical threshold.

Solution:
1. Check disk usage: `df -h`
2. Identify large files: `du -sh /*`
3. Delete unnecessary files: `rm -rf /path/to/unneeded/files`
4. Clean up unused Docker images: `docker system prune`
5. Monitor usage regularly.
1: Pod default/brokenpod()
- Error: Back-off pulling image "nginx:i_dont_exist": ErrImagePull: rpc error: code = NotFound desc = failed to pull and unpack image "docker.io/library/nginx:i_dont_exist": failed to resolve image: docker.io/library/nginx:i_dont_exist: not found
Error: The specified Docker image "nginx:i_dont_exist" cannot be found in the repository.

Solution:
1. Check the image name for typos.
2. Verify the image tag exists (e.g., use "nginx:latest").
3. If using a private repository, ensure proper authentication.
4. Retry pulling the image with the correct name.
```

You can learn more here https://docs.k8sgpt.ai/tutorials/custom-analyzers/.


## MCP


### Codex CLI via ChatGPT

```
#$ brew install codex
$ sudo apt install bubblewrap # needed on Ubuntu 24.04+
$ curl -fsSL https://chatgpt.com/codex/install.sh | sh
$ export PATH=$HOME/.local/bin:$PATH

$ mkdir ~/codex_k8sgpt_via_mcp && cd ~/codex_k8sgpt_via_mcp
```

```
$ export OPENAI_API_KEY=sk-proj-...

#$ codex login status
$ codex login status

Not logged in

$ printenv OPENAI_API_KEY | codex login --with-api-key

Reading API key from stdin...
Successfully logged in

$ codex login status

Logged in using an API key - sk-proj-***DlVYA
```

Now we integrate K8sGPT via an MCP configuration (using STDIO).

```
$ codex mcp list

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
> You are in /home/ubuntu/codex_k8sgpt_via_mcp

  Do you trust the contents of this directory? Working with untrusted contents comes with higher risk of prompt injection. Trusting the directory allows project-local config, hooks, and
  exec policies to load.

› 1. Yes, continue
  2. No, quit

  Press enter to continue
```

* Press "enter"

```
╭───────────────────────────────────────────╮
│ >_ OpenAI Codex (v0.147.0)                │
│                                           │
│ model:     gpt-5.6-sol   /model to change │
│ directory: ~/codex_k8sgpt_via_mcp         │
╰───────────────────────────────────────────╯

  Tip: Our most capable model yet. GPT-5.6 Sol can tackle complex code changes, dig into research, produce polished documents, and take on your most ambitious work. Sol is highly capable
  at lower reasoning efforts—try starting lower, then turn it up for harder jobs.


› Run /review on my current changes

  gpt-5.6-sol default · ~/codex_k8sgpt_via_mcp
```

List MCP services.

```
/mcp

🔌  MCP Tools

  • No MCP tools available.

  • k8sgpt
    • Auth: Unsupported
    • Tools: (none)



› Run /review on my current changes

  gpt-5.6-sol default · ~/codex_k8sgpt_via_mcp
```

We see the available tools and resources. Lets call "list-namespaces".

```
› #list-namespaces


• I’ll query the connected Kubernetes tooling and return the available namespaces.

◦ Calling k8sgpt.list-namespaces({})


  Field 1/1
  Allow the k8sgpt MCP server to run tool "list-namespaces"?
  › 1. Allow                   Run the tool and continue.
    2. Allow for this session  Run the tool and remember this choice for this session.
    3. Always allow            Run the tool and remember this choice for future tool calls.
    4. Cancel                  Cancel this tool call
  enter to submit | esc to cancel
```

```
› #list-namespaces


• I’ll query the connected Kubernetes tooling and return the available namespaces.

• Called k8sgpt.list-namespaces({})
  └ [{"metadata": {"name": "cilium-secrets", "uid": "35fa833b-b5f5-4933-8134-a78ab9972a27", "resourceVersion": "336", "creationTimestamp": "2026-08-12T00:18:47Z", "labels":
        {"app.kubernetes.io/managed-by": "Helm", "app.kubernetes.io/part-of": "cilium", "helm.sh/chart": "cilium-1.20.0", "kubernetes.io/metadata.name": "cilium-secrets"}, "annotations":
        {"meta.helm.sh/release-name": "cilium", "meta.helm.sh/release-namespace": "kube-system"}, "managedFields": [{"manager": "cilium", "operation": "Update", "apiVersion": "v1", "time":
        "2026-08-12T00:18:47Z", "fieldsType": "FieldsV1", "fieldsV1": {"f:metadata": {"f:annotations": {".": {}, "f:meta.helm.sh/release-name": {}, "f:meta.helm.sh/release-namespace": {}},
        "f:labels": {".": {}, "f:app.kubernetes.io/managed-by": {}, "f:app.kubernetes.io/part-of": {}, "f:helm.sh/chart": {}, "f:kubernetes.io/metadata.name": {}}}}}]}, "spec":
        {"finalizers": ["kubernetes"]}, "...

────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────

• The cluster query is still in progress; I’m waiting for the Kubernetes API response.

────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────

• Active namespaces:

  - cilium-secrets
  - default
  - kube-node-lease
  - kube-public
  - kube-system

────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────


› Run /review on my current changes

  gpt-5.6-sol default · ~/codex_k8sgpt_via_mcp
```

Now lets "analyze"

```
› #analyze


• I’ll analyze Kubernetes resources across all namespaces and return the detected issues.

◦ Calling k8sgpt.analyze({"namespace":"","explain":true})


  Field 1/1
  Allow the k8sgpt MCP server to run tool "analyze"?

  explain: true
  namespace:

  › 1. Allow                   Run the tool and continue.
    2. Allow for this session  Run the tool and remember this choice for this session.
    3. Always allow            Run the tool and remember this choice for future tool calls.
    4. Cancel                  Cancel this tool call
  enter to submit | esc to cancel
```

```
› #analyze


• I’ll analyze Kubernetes resources across all namespaces and return the detected issues.

• Called k8sgpt.analyze({"namespace":"","explain":true})
  └ AI Provider: openai

    0: Pod default/brokenpod()
    - Error: Back-off pulling image "nginx:i_dont_exist": ErrImagePull: rpc error: code = NotFound desc = failed to pull and unpack image "docker.io/library/nginx:i_dont_exist": failed to
        resolve image: docker.io/library/nginx:i_dont_exist: not found
    Error: The error indicates that Kubernetes is unable to pull the specified Docker image "nginx:i_dont_exist" because it does not exist in the Docker registry.

    Solution:
    1. Check the image name for typos.
    2. Verify the image tag exists on Docker Hub.
    3. If using a private registry, ensure proper authentication.
    4. Update the deployment with a valid image name/tag.
    5. Redeploy the application.
    1: ConfigMap cilium-secrets/kube-root-ca.crt()
    - Error: ConfigMap kube-root-ca.crt is not used by any pods in the namespace
    Error: The ConfigMap `kube-root-ca.crt` is not being utilized by any pods in the specified names...

────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────

• Found 11 findings across all namespaces:

  - Action required: default/brokenpod cannot pull nginx:i_dont_exist because that image tag does not exist. Update it to a valid tag, such as nginx:latest.
  - 10 low-confidence warnings: ConfigMaps reported as unused.

  Avoid deleting system ConfigMaps such as kube-root-ca.crt, cluster-info, kubeadm-config, or kubelet-config; these warnings are expected false positives because they may be consumed by
  Kubernetes components rather than mounted by pods.

────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────


› Run /review on my current changes

  gpt-5.6-sol default · ~/codex_k8sgpt_via_mcp
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



TODO - Fix Codex listing issue


```
# stop codex or run
$ k8sgpt serve --mcp --mcp-http --mcp-port 8089

# another terminal
$ curl -s -X POST http://localhost:8089/mcp -H 'Content-Type: application/json' -d '{"jsonrpc":"2.0","id":1,"method":"tools/list"}'

...
```

```
# if codex stopped
$ sudo apt install npm
$ npx -y @modelcontextprotocol/inspector@v1-latest   --cli k8sgpt serve --mcp --method tools/list

⚠️  @modelcontextprotocol/inspector v1 is deprecated.
    Upgrade:  npx @modelcontextprotocol/inspector@latest
    v1 receives security fixes only.

{
  "tools": [
    {
      "name": "add-filters",
      "description": "Add filters to enable specific analyzers",
      "inputSchema": {
        "type": "object",
        "properties": {
          "filters": {
            "description": "List of filter names to add (e.g., ['Pod', 'Service', 'Deployment'])",
            "items": {
              "type": "string"
            },
            "type": "array"
          }
        },
        "required": [
          "filters"
        ]
      },
      "annotations": {
        "readOnlyHint": false,
        "destructiveHint": true,
        "idempotentHint": false,
        "openWorldHint": true
      }
    },
    {
      "name": "analyze",
      "description": "Analyze Kubernetes resources for issues and problems",
      "inputSchema": {
        "type": "object",
        "properties": {
          "backend": {
            "description": "AI backend to use for analysis (e.g., openai, azure, localai)",
            "type": "string"
          },
          "explain": {
            "description": "Provide detailed explanations for issues",
            "type": "boolean"
          },
          "filters": {
            "description": "Provide filters to narrow down the analysis (e.g. ['Pods', 'Deployments'])",
            "items": {
              "type": "string"
            },
            "type": "array"
          },
          "namespace": {
            "description": "Kubernetes namespace to analyze (empty for all namespaces)",
            "type": "string"
          }
        }
      },
      "annotations": {
        "readOnlyHint": false,
        "destructiveHint": true,
        "idempotentHint": false,
        "openWorldHint": true
      }
    },
    {
      "name": "cluster-info",
      "description": "Get Kubernetes cluster information and version",
      "inputSchema": {
        "type": "object",
        "properties": {}
      },
      "annotations": {
        "readOnlyHint": false,
        "destructiveHint": true,
        "idempotentHint": false,
        "openWorldHint": true
      }
    },
    {
      "name": "config",
      "description": "Configure K8sGPT settings including custom analyzers and cache",
      "inputSchema": {
        "type": "object",
        "properties": {
          "cache": {
            "description": "Cache configuration",
            "properties": {
              "bucketName": {
                "description": "Bucket name for S3/GCS cache",
                "type": "string"
              },
              "containerName": {
                "description": "Container name for Azure cache",
                "type": "string"
              },
              "endpoint": {
                "description": "Custom endpoint for S3 cache",
                "type": "string"
              },
              "insecure": {
                "description": "Use insecure connection for cache",
                "type": "boolean"
              },
              "projectId": {
                "description": "Project ID for GCS cache",
                "type": "string"
              },
              "region": {
                "description": "Region for S3/GCS cache",
                "type": "string"
              },
              "storageAccount": {
                "description": "Storage account for Azure cache",
                "type": "string"
              },
              "type": {
                "description": "Cache type (s3, azure, gcs)",
                "enum": [
                  "s3",
                  "azure",
                  "gcs"
                ],
                "type": "string"
              }
            },
            "type": "object"
          },
          "customAnalyzers": {
            "description": "Custom analyzer configurations",
            "properties": {
              "connection": {
                "properties": {
                  "port": {
                    "description": "Port of the custom analyzer service",
                    "type": "integer"
                  },
                  "url": {
                    "description": "URL of the custom analyzer service",
                    "type": "string"
                  }
                },
                "type": "object"
              },
              "name": {
                "description": "Name of the custom analyzer",
                "type": "string"
              }
            },
            "type": "object"
          }
        }
      },
      "annotations": {
        "readOnlyHint": false,
        "destructiveHint": true,
        "idempotentHint": false,
        "openWorldHint": true
      }
    },
    {
      "name": "get-logs",
      "description": "Get logs from a pod container",
      "inputSchema": {
        "type": "object",
        "properties": {
          "container": {
            "description": "Container name (if pod has multiple containers)",
            "type": "string"
          },
          "namespace": {
            "description": "Namespace of the pod",
            "type": "string"
          },
          "podName": {
            "description": "Name of the pod",
            "type": "string"
          },
          "previous": {
            "description": "Get logs from previous terminated container",
            "type": "boolean"
          },
          "sinceSeconds": {
            "description": "Return logs newer than this many seconds",
            "type": "number"
          },
          "tailLines": {
            "description": "Number of lines from the end of logs (default: 100)",
            "type": "number"
          }
        },
        "required": [
          "podName",
          "namespace"
        ]
      },
      "annotations": {
        "readOnlyHint": false,
        "destructiveHint": true,
        "idempotentHint": false,
        "openWorldHint": true
      }
    },
    {
      "name": "get-resource",
      "description": "Get detailed information about a specific Kubernetes resource",
      "inputSchema": {
        "type": "object",
        "properties": {
          "name": {
            "description": "Name of the resource",
            "type": "string"
          },
          "namespace": {
            "description": "Namespace of the resource (required for namespaced resources)",
            "type": "string"
          },
          "resourceType": {
            "description": "Type of resource (e.g., pod, deployment, service)",
            "type": "string"
          }
        },
        "required": [
          "resourceType",
          "name"
        ]
      },
      "annotations": {
        "readOnlyHint": false,
        "destructiveHint": true,
        "idempotentHint": false,
        "openWorldHint": true
      }
    },
    {
      "name": "list-events",
      "description": "List Kubernetes events for debugging and troubleshooting",
      "inputSchema": {
        "type": "object",
        "properties": {
          "involvedObjectKind": {
            "description": "Filter events by involved object kind (e.g., Pod, Deployment)",
            "type": "string"
          },
          "involvedObjectName": {
            "description": "Filter events by involved object name (e.g., pod name)",
            "type": "string"
          },
          "limit": {
            "description": "Maximum number of events to return (default: 100)",
            "type": "number"
          },
          "namespace": {
            "description": "Namespace to list events from (empty for all namespaces)",
            "type": "string"
          }
        }
      },
      "annotations": {
        "readOnlyHint": false,
        "destructiveHint": true,
        "idempotentHint": false,
        "openWorldHint": true
      }
    },
    {
      "name": "list-filters",
      "description": "List all available and active analyzers/filters in k8sgpt",
      "inputSchema": {
        "type": "object",
        "properties": {}
      },
      "annotations": {
        "readOnlyHint": false,
        "destructiveHint": true,
        "idempotentHint": false,
        "openWorldHint": true
      }
    },
    {
      "name": "list-integrations",
      "description": "List available integrations (Prometheus, AWS, Keda, Kyverno, etc.)",
      "inputSchema": {
        "type": "object",
        "properties": {}
      },
      "annotations": {
        "readOnlyHint": false,
        "destructiveHint": true,
        "idempotentHint": false,
        "openWorldHint": true
      }
    },
    {
      "name": "list-namespaces",
      "description": "List all namespaces in the cluster",
      "inputSchema": {
        "type": "object",
        "properties": {}
      },
      "annotations": {
        "readOnlyHint": false,
        "destructiveHint": true,
        "idempotentHint": false,
        "openWorldHint": true
      }
    },
    {
      "name": "list-resources",
      "description": "List Kubernetes resources of a specific type (pods, deployments, services, nodes, etc.)",
      "inputSchema": {
        "type": "object",
        "properties": {
          "labelSelector": {
            "description": "Label selector to filter resources (e.g., 'app=myapp')",
            "type": "string"
          },
          "namespace": {
            "description": "Namespace to list resources from (empty for all or cluster-scoped resources)",
            "type": "string"
          },
          "resourceType": {
            "description": "Type of resource to list (e.g., pods, deployments, services, nodes, jobs, etc.)",
            "type": "string"
          }
        },
        "required": [
          "resourceType"
        ]
      },
      "annotations": {
        "readOnlyHint": false,
        "destructiveHint": true,
        "idempotentHint": false,
        "openWorldHint": true
      }
    },
    {
      "name": "remove-filters",
      "description": "Remove filters to disable specific analyzers",
      "inputSchema": {
        "type": "object",
        "properties": {
          "filters": {
            "description": "List of filter names to remove",
            "items": {
              "type": "string"
            },
            "type": "array"
          }
        },
        "required": [
          "filters"
        ]
      },
      "annotations": {
        "readOnlyHint": false,
        "destructiveHint": true,
        "idempotentHint": false,
        "openWorldHint": true
      }
    }
  ]
}
```

```
$ codex doctor --json | jq '.checks["mcp.config"]'
{
  "id": "mcp.config",
  "category": "mcp",
  "status": "ok",
  "summary": "MCP configuration is locally consistent",
  "details": {
    "configured servers": "1",
    "disabled servers": "0",
    "stdio servers": "1"
  },
  "remediation": null,
  "durationMs": 0
}
```

example how flaky

```
› #tools/list


• I’m using the OpenAI docs skill because this asks about the tools available in the current Codex session.

• Explored
  └ Read SKILL.md (openai-docs skill)

• Searching the web

• Searched the web for site:developers.openai.com/codex tools list Codex

────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────

• Available tools:

  - Shell command execution and stdin control
  - File patching
  - Local image viewing
  - Web search, pages, weather, finance, sports, and time
  - MCP resource listing and reading
  - Goal and plan management
  - Agent collaboration/delegation
  - Image generation skill
  - Codex/OpenAI docs skill
  - Plugin and skill creation/installation

  Some tools require approval for elevated filesystem or network access.

────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────

/mcp

🔌  MCP Tools

  • No MCP tools available.

  • k8sgpt
    • Auth: Unsupported
    • Tools: (none)



› #tools/list


• I’m checking the current session registry so the list reflects the tools enabled right now. I’m also applying the OpenAI docs skill because this is a Codex capability query.

• Explored
  └ Read SKILL.md (openai-docs skill)

────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────

• K8sGPT MCP tools:

  - add_filters
  - analyze
  - cluster_info
  - config
  - get_logs
  - get_resource
  - list_events
  - list_filters
  - list_integrations
  - list_namespaces
  - list_resources
  - remove_filters

  Also available: shell execution, file patching, web access, MCP resource discovery, image viewing, planning/goals, and agent collaboration.

────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────


› #prompts/get


• prompts/get is unavailable: the connected K8sGPT MCP server exposes tools, but no MCP prompts.


› #troubleshoot-pod


• Provide the pod name and namespace, for example:

  #troubleshoot-pod pod=my-app-abc123 namespace=default


› Summarize recent commits

  gpt-5.6-sol default · ~/codex_k8sgpt_via_mcp
```
