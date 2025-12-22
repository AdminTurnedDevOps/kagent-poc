# Kagent POC

This POC contains test cases around:
1. MCP connectivity/security
2. Agentic AI (creating/managing Agents) and Agent security
3. Agentic routing (AI Gateway) 
4. Custom LLMs/Model implementation

## MCP Connectivity

### MCP Server Connection

1. . Deploy the MCP Server (Streamable HTTP)
```
kubectl apply -f - <<EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: mcp-math-script
  namespace: default
data:
  server.py: |
    import uvicorn
    from mcp.server.fastmcp import FastMCP
    from starlette.applications import Starlette
    from starlette.routing import Route
    from starlette.requests import Request
    from starlette.responses import JSONResponse

    mcp = FastMCP("Math-Service")

    @mcp.tool()
    def add(a: int, b: int) -> int:
        return a + b

    @mcp.tool()
    def multiply(a: int, b: int) -> int:
        return a * b

    async def handle_mcp(request: Request):
        try:
            data = await request.json()
            method = data.get("method")
            msg_id = data.get("id")
            result = None
            
            if method == "initialize":
                result = {
                    "protocolVersion": "2024-11-05",
                    "capabilities": {"tools": {}},
                    "serverInfo": {"name": "Math-Service", "version": "1.0"}
                }
            
            elif method == "notifications/initialized":
                return JSONResponse({"jsonrpc": "2.0", "id": msg_id, "result": True})

            elif method == "tools/list":
                tools_list = await mcp.list_tools()
                result = {
                    "tools": [
                        {
                            "name": t.name,
                            "description": t.description,
                            "inputSchema": t.inputSchema
                        } for t in tools_list
                    ]
                }

            elif method == "tools/call":
                params = data.get("params", {})
                name = params.get("name")
                args = params.get("arguments", {})
                
                # Call the tool
                tool_result = await mcp.call_tool(name, args)
                
                # --- FIX: Serialize the content objects manually ---
                serialized_content = []
                for content in tool_result:
                    if hasattr(content, "type") and content.type == "text":
                        serialized_content.append({"type": "text", "text": content.text})
                    elif hasattr(content, "type") and content.type == "image":
                         serialized_content.append({
                             "type": "image", 
                             "data": content.data, 
                             "mimeType": content.mimeType
                         })
                    else:
                        # Fallback for dictionaries or other types
                        serialized_content.append(content if isinstance(content, dict) else str(content))

                result = {
                    "content": serialized_content,
                    "isError": False
                }

            elif method == "ping":
                result = {}

            else:
                return JSONResponse(
                    {"jsonrpc": "2.0", "id": msg_id, "error": {"code": -32601, "message": "Method not found"}},
                    status_code=404
                )

            return JSONResponse({"jsonrpc": "2.0", "id": msg_id, "result": result})

        except Exception as e:
            # Print error to logs for debugging
            import traceback
            traceback.print_exc()
            return JSONResponse(
                {"jsonrpc": "2.0", "id": None, "error": {"code": -32603, "message": str(e)}},
                status_code=500
            )

    app = Starlette(routes=[
        Route("/mcp", handle_mcp, methods=["POST"]),
        Route("/", lambda r: JSONResponse({"status": "ok"}), methods=["GET"])
    ])

    if __name__ == "__main__":
        print("Starting Fixed Math Server on port 8000...")
        uvicorn.run(app, host="0.0.0.0", port=8000)
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mcp-math-server
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mcp-math-server
  template:
    metadata:
      labels:
        app: mcp-math-server
    spec:
      containers:
      - name: math
        image: python:3.11-slim
        command: ["/bin/sh", "-c"]
        args:
        - |
          pip install "mcp[cli]" uvicorn starlette && 
          python /app/server.py
        ports:
        - containerPort: 8000
        volumeMounts:
        - name: script-volume
          mountPath: /app
        readinessProbe:
          httpGet:
            path: /
            port: 8000
          initialDelaySeconds: 5
          periodSeconds: 5
      volumes:
      - name: script-volume
        configMap:
          name: mcp-math-script
---
apiVersion: v1
kind: Service
metadata:
  name: mcp-math-server
  namespace: default
spec:
  selector:
    app: mcp-math-server
  ports:
  - port: 80
    targetPort: 8000
EOF
```

2. Create the Gateway
```
kubectl apply -f - <<EOF
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: agentgateway
  namespace: gloo-system
spec:
  gatewayClassName: agentgateway-enterprise
  listeners:
  - name: http
    port: 8080
    protocol: HTTP
    allowedRoutes:
      namespaces:
        from: Same
EOF
```

3. Create the Backend
```
kubectl apply -f - <<EOF
apiVersion: gateway.kgateway.dev/v1alpha1
kind: Backend
metadata:
  name: mcp-backend
  namespace: gloo-system
spec:
  type: MCP
  mcp:
    targets:
    - name: math-mcp-server
      static:
        host: mcp-math-server.default.svc.cluster.local
        port: 80
        protocol: StreamableHTTP
EOF
```


4. Create the HTTPRoute
```
kubectl apply -f - <<EOF
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: mcp-route
  namespace: gloo-system
spec:
  parentRefs:
  - name: agentgateway
  rules:
  - backendRefs:
    - name: mcp-backend
      group: gateway.kgateway.dev
      kind: Backend
EOF
```

5. Test the Gateway/route
```
export GATEWAY_IP=$(kubectl get svc agentgateway -n gloo-system -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
echo $GATEWAY_IP
```

6. Open MCP Inspector
```
npx modelcontextprotocol/inspector#0.16.2
```

URL to put into Inspector: `http://YOUR_ALB_LB_IP:8080/mcp`


### Secure Connectivity To An MCP Server (StreamableHTTP)

1. Add in a traffic policy for auth based on a JWT token
```
kubectl apply -f- <<EOF
apiVersion: gloo.solo.io/v1alpha1
kind: GlooTrafficPolicy
metadata:
  name: jwt
  namespace: gloo-system
spec:
  targetRefs:
    - group: gateway.networking.k8s.io
      kind: Gateway
      name: agentgateway
  glooJWT:
    beforeExtAuth:
      providers:
        selfminted:
          issuer: solo.io
          jwks:
            local:
              key: '{"keys":[{"kty":"RSA","kid":"solo-public-key-001","use":"sig","alg":"RS256","n":"AOfIaJMUm7564sWWNHaXt_hS8H0O1Ew59-nRqruMQosfQqa7tWne5lL3m9sMAkfa3Twx0LMN_7QqRDoztvV3Wa_JwbMzb9afWE-IfKIuDqkvog6s-xGIFNhtDGBTuL8YAQYtwCF7l49SMv-GqyLe-nO9yJW-6wIGoOqImZrCxjxXFzF6mTMOBpIODFj0LUZ54QQuDcD1Nue2LMLsUvGa7V1ZHsYuGvUqzvXFBXMmMS2OzGir9ckpUhrUeHDCGFpEM4IQnu-9U8TbAJxKE5Zp8Nikefr2ISIG2Hk1K2rBAc_HwoPeWAcAWUAR5tWHAxx-UXClSZQ9TMFK850gQGenUp8","e":"AQAB"}]}'
EOF
```

2. Save the token for "Bob"
```
eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCIsImtpZCI6InNvbG8tcHVibGljLWtleS0wMDEifQ.eyJpc3MiOiJzb2xvLmlvIiwib3JnIjoic29sby5pbyIsInN1YiI6ImJvYiIsInRlYW0iOiJvcHMiLCJleHAiOjIwNzQyNzQ5NTQsImxsbXMiOnsibWlzdHJhbGFpIjpbIm1pc3RyYWwtbGFyZ2UtbGF0ZXN0Il19fQ.GF_uyLpZSTT1DIvJeO_eish1WDjMaS4BQSifGQhqPRLjzu3nXtPkaBRjceAmJi9gKZYAzkT25MIrT42ZIe3bHilrd1yqittTPWrrM4sWDDeldnGsfU07DWJHyboNapYR-KZGImSmOYshJlzm1tT_Bjt3-RK3OBzYi90_wl0dyAl9D7wwDCzOD4MRGFpoMrws_OgVrcZQKcadvIsH8figPwN4mK1U_1mxuL08RWTu92xBcezEO4CdBaFTUbkYN66Y2vKSTyPCxg3fLtg1mvlzU1-Wgm2xZIiPiarQHt6Uq7v9ftgzwdUBQM1AYLvUVhCN6XkkR9OU3p0OXiqEDjAxcg
```

4. Open MCP Inspector and try to re-connect to the MCP Server. You'll see something similar to the below:
```
Connection Error - Check if your MCP server is running and proxy token is correct
```

4. Click on **Authentication**
- Header Name: **Authorization**
- Bearer Token: Bobs Token from step 2

### Specify MCP Server Tool List

You can specify what tools from an MCP Server you want exposed via your AI Gateway

1. Create a policy that specifies no tools listed
```
kubectl apply -f- <<EOF
apiVersion: gloo.solo.io/v1alpha1
kind: GlooTrafficPolicy
metadata:
  name: jwt-rbac
  namespace: gloo-system
spec:
  targetRefs:
    - group: gateway.kgateway.dev
      kind: Backend
      name: mcp-backend
  rbac:
    policy:
      matchExpressions:
        - 'mcp.tool.name == ""'
EOF
```

2. Disconnect and reconnect via the MCP Inspector and you should see no tools

3. Update the policy to include the **multiply** tool

```
kubectl apply -f- <<EOF
apiVersion: gloo.solo.io/v1alpha1
kind: GlooTrafficPolicy
metadata:
  name: jwt-rbac
  namespace: gloo-system
spec:
  targetRefs:
    - group: gateway.kgateway.dev
      kind: Backend
      name: mcp-backend
  rbac:
    policy:
      matchExpressions:
        - 'mcp.tool.name == "multiply"'
EOF
```

### Connecting To Remote MCP Servers

1. Create the github pat token environment variable:
```
export GITHUB_PERSONAL_ACCESS_TOKEN=your_github_pat_here
```

2. Create the k8s secret to store the PAT token:
```
kubectl apply -f - <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: github-pat
  namespace: kagent
type: Opaque
stringData:
  GITHUB_PERSONAL_ACCESS_TOKEN: $GITHUB_PERSONAL_ACCESS_TOKEN
EOF
```

3. Create a new MCP Server object in Kubernetes:
```
kubectl apply -f - <<EOF
apiVersion: kagent.dev/v1alpha2
kind: RemoteMCPServer
metadata:
  name: github-mcp-remote
  namespace: kagent
spec:
  description: GitHub Copilot MCP Server
  url: https://api.githubcopilot.com/mcp/
  protocol: STREAMABLE_HTTP
  headersFrom:
    - name: Authorization
      valueFrom:
        type: Secret
        name: github-pat
        key: GITHUB_PERSONAL_ACCESS_TOKEN
  timeout: 5s
  terminateOnClose: true
EOF
```

4. Create a new Agent and use the MCP Server you created in the previous step:
```
kubectl apply -f - <<EOF
apiVersion: kagent.dev/v1alpha2
kind: Agent
metadata:
  name: github-mcp-agent
  namespace: kagent
spec:
  description: This agent can interact with GitHub repositories, issues, pull requests, and more
  type: Declarative
  declarative:
    modelConfig: default-model-config
    systemMessage: |-
      You're a friendly and helpful agent that uses GitHub tools to help with repository management, issues, pull requests, and code review

      # Instructions

      - If user question is unclear, ask for clarification before running any tools
      - Always be helpful and friendly
      - If you don't know how to answer the question DO NOT make things up
        respond with "Sorry, I don't know how to answer that" and ask the user to further clarify the question

      # Response format
      - ALWAYS format your response as Markdown
      - Your response will include a summary of actions you took and an explanation of the result
    tools:
    - type: McpServer
      mcpServer:
        name: github-mcp-remote
        kind: RemoteMCPServer
        toolNames:
        - get_latest_release
        - get_commit
        - get_tag
        - list_branches
EOF
```

4. Look at the Agent configuration and wait until both `READY` and `ACCEPTED` are in a `True` status:
```
kubectl get agents -n kagent
```

5. Open the kagent UI, go to the UI, and ask: `What branches are available under `https://github.com/AdminTurnedDevOps/agentic-demo-repo``


## Agentic Routing and Agent Creation/Management


### Route Agent Traffic Through Agentgateway

prereq: Ensure you have an Anthropic Secret in the `kagent` namespace
```
kubectl apply -f- <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: anthropic-secret
  namespace: kagent
  labels:
    app: agentgateway
type: Opaque
stringData:
  Authorization: $ANTHROPIC_API_KEY
EOF
```

1. Create the Gateway
```
kubectl apply -f- <<EOF
kind: Gateway
apiVersion: gateway.networking.k8s.io/v1
metadata:
  name: agentgateway-route
  namespace: gloo-system
  labels:
    app: agentgateway-route
spec:
  gatewayClassName: agentgateway-enterprise
  listeners:
  - protocol: HTTP
    port: 8080
    name: http
    allowedRoutes:
      namespaces:
        from: All
EOF
```

2. Retrieve ALB public IP
```
export INGRESS_GW_ADDRESS=$(kubectl get svc -n gloo-system agentgateway-route -o jsonpath="{.status.loadBalancer.ingress[0]['hostname','ip']}")
echo $INGRESS_GW_ADDRESS
```

3. Create the Backend object so agentgateway knows what to route to

```
kubectl apply -f- <<EOF
apiVersion: gateway.kgateway.dev/v1alpha1
kind: Backend
metadata:
  labels:
    app: agentgateway-route
  name: anthropic-routing-backend
  namespace: gloo-system
spec:
  type: AI
  ai:
    llm:
        anthropic:
          authToken:
            kind: SecretRef
            secretRef:
              name: anthropic-secret
          model: "claude-3-5-haiku-latest"
EOF
```

4. Create the route

```
kubectl apply -f- <<EOF
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: claude
  namespace: gloo-system
  labels:
    app: agentgateway-route
spec:
  parentRefs:
    - name: agentgateway-route
      namespace: gloo-system
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /anthropic
    filters:
    - type: URLRewrite
      urlRewrite:
        path:
          type: ReplaceFullPath
          replaceFullPath: /v1/chat/completions
    backendRefs:
    - name: anthropic-routing-backend
      namespace: gloo-system
      group: gateway.kgateway.dev
      kind: Backend
EOF
```

5. Create Model Config that will be used by the new agent.

```
kubectl apply -f - <<EOF
apiVersion: kagent.dev/v1alpha2
kind: ModelConfig
metadata:
  name: anthropic-model-config
  namespace: kagent
spec:
  apiKeySecret: anthropic-secret
  apiKeySecretKey: Authorization
  model: claude-3-5-haiku-latest
  provider: OpenAI
  openAI:
    baseUrl: http://YOUR_GATEWAY_IP_FROM_STEP2:8080/anthropic
EOF
```

6. Create the agent
```
kubectl apply -f - <<EOF
apiVersion: kagent.dev/v1alpha2
kind: Agent
metadata:
  name: testing-agentgateway
  namespace: kagent
spec:
  description: This agent can use a single tool to expand it's Kubernetes knowledge for troubleshooting and deployment
  type: Declarative
  declarative:
    modelConfig: anthropic-model-config
    systemMessage: |-
      You're a friendly and helpful agent that uses the Kubernetes tool to help troubleshooting and deploy environments
  
      # Instructions
  
      - If user question is unclear, ask for clarification before running any tools
      - Always be helpful and friendly
      - If you don't know how to answer the question DO NOT make things up
        respond with "Sorry, I don't know how to answer that" and ask the user to further clarify the question
  
      # Response format
      - ALWAYS format your response as Markdown
      - Your response will include a summary of actions you took and an explanation of the result
EOF
```

7. Open the `testing-agentgateway` Agent in kagent and prompt it with the following:
```
What can you do?
```


8. Run `kubectl logs agentgateway-POD-NAME -n gloo-system` and you should see an output similar to the below:

```
2025-12-05T17:18:05.241265Z     info    request gateway=gloo-system/agentgateway listener=http route=gloo-system/claude endpoint=api.anthropic.com:443 src.addr=192.168.26.166:30892 http.method=POST http.host=xxxx3xxxxx-34xxxxx7.us-east-1.elb.amazonaws.com http.path=/anthropic/chat/completions http.version=HTTP/1.1 http.status=200 protocol=llm gen_ai.operation.name=chat gen_ai.provider.name=anthropic gen_ai.request.model=claude-3-5-haiku-latest gen_ai.response.model=claude-3-5-haiku-20241022 gen_ai.usage.input_tokens=182 gen_ai.usage.output_tokens=269 duration=5929ms
```

9. You can also test via a `curl` on your local terminal
```
curl "$INGRESS_GW_ADDRESS:8080/anthropic" -H content-type:application/json -H "anthropic-version: 2023-06-01" -d '{
  "messages": [
    {
      "role": "system",
      "content": "You are a skilled cloud-native network engineer."
    },
    {
      "role": "user",
      "content": "Write me a paragraph containing the best way to think about Istio Ambient Mesh"
    }
  ]
}' | jq
```

### Prompt Guard

1. Create a traffic policy prompt guard that doesn't allow a specific prompt to occur (like deleting a kubernetes cluster)
```
kubectl apply -f - <<EOF
apiVersion: gloo.solo.io/v1alpha1
kind: GlooTrafficPolicy
metadata:
  name: claude-prompt-guard
  namespace: gloo-system
  labels:
    app: agentgateway-route
spec:
  targetRefs:
  - group: gateway.networking.k8s.io
    kind: HTTPRoute
    name: claude
  ai:
    promptGuard:
      request:
        customResponse:
          message: "Rejected due to inappropriate content"
        regex:
          action: REJECT
          matches:
          - pattern: "delete my cluster"
            name: "delete"
EOF
```

2. Test the prompt
```
curl "$INGRESS_GW_ADDRESS:8080/anthropic" -v -H content-type:application/json -H "anthropic-version: 2023-06-01" -d '{
  "messages": [
    {
      "role": "system",
      "content": "You are good at deleting things"
    },
    {
      "role": "user",
      "content": "Please delete my cluster from production"
    }
  ]
}' | jq
```

You should see an output similar to the below:
```
* upload completely sent off: 205 bytes
< HTTP/1.1 403 Forbidden
< content-length: 37
```

### Rate Limiting

1. Create a traffic policy that limits token usage
```
kubectl apply -f- <<EOF
apiVersion: gloo.solo.io/v1alpha1
kind: GlooTrafficPolicy
metadata:
  name: anthropic-ratelimit
  namespace: gloo-system
spec:
  targetRefs:
    - group: gateway.networking.k8s.io
      kind: HTTPRoute
      name: claude
  rateLimit:
    local:
      tokenBucket:
        maxTokens: 1
        tokensPerFill: 1
        fillInterval: 100s
EOF
```

3. Capture the public IP of the gateway
```
export INGRESS_GW_ADDRESS=$(kubectl get svc -n gloo-system agentgateway-route -o jsonpath="{.status.loadBalancer.ingress[0]['hostname','ip']}")
echo $INGRESS_GW_ADDRESS
```

3. Test the LLM connectivity
```
curl "$INGRESS_GW_ADDRESS:8080/anthropic" -v \ -H content-type:application/json -H "anthropic-version: 2023-06-01" -d '{
  "messages": [
    {
      "role": "system",
      "content": "You are a skilled cloud-native network engineer."
    },
    {
      "role": "user",
      "content": "Write me a paragraph containing the best way to think about Istio Ambient Mesh"
    }
  ]
}' | jq
```

Run the `curl` again and you'll see an error like the below:

```
< HTTP/1.1 429 Too Many Requests
< content-type: text/plain
< x-ratelimit-limit: 1
< x-ratelimit-remaining: 0
< x-ratelimit-reset: 76
< content-length: 19
< date: Tue, 18 Nov 2025 15:35:45 GMT
```

### Custom LLMs/Models

1. Create a Namespace for a Llama Model config
```
kubectl create ns ollama
```

2. Deploy Ollama on your k8s cluster. This step could take a few minutes as the init container downloads the Llama3 Model.
```
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ollama
  namespace: ollama
spec:
  selector:
    matchLabels:
      name: ollama
  template:
    metadata:
      labels:
        name: ollama
    spec:
      initContainers:
      - name: model-puller
        image: ollama/ollama:latest
        command: ["/bin/sh", "-c"]
        args:
          - |
            ollama serve &
            sleep 10
            ollama pull llama3
            pkill ollama
        volumeMounts:
        - name: ollama-data
          mountPath: /root/.ollama
        resources:
          requests:
            memory: "8Gi"
          limits:
            memory: "12Gi"
      containers:
      - name: ollama
        image: ollama/ollama:latest
        ports:
        - name: http
          containerPort: 11434
          protocol: TCP
        volumeMounts:
        - name: ollama-data
          mountPath: /root/.ollama
        resources:
          requests:
            memory: "8Gi"
          limits:
            memory: "12Gi"
      volumes:
      - name: ollama-data
        emptyDir: {}
---
apiVersion: v1
kind: Service
metadata:
  name: ollama
  namespace: ollama
spec:
  type: ClusterIP
  selector:
    name: ollama
  ports:
  - port: 80
    name: http
    targetPort: http
    protocol: TCP
EOF
```

3. Confirm that the Model was downloaded:
```
kubectl exec -n ollama deployment/ollama -- ollama list
```

You should see an output similar to the one below:
```
Defaulted container "ollama" out of: ollama, model-puller (init)
NAME             ID              SIZE      MODIFIED           
llama3:latest    365c0bd3c000    4.7 GB    About a minute ago
```

4. Ensure that everything was deployed as expected
```
kubectl get all -n ollama
```

5. Create a `ModelConfig` that will add the Llama Model to the list of Models in kagent
```
kubectl apply -f - <<EOF
apiVersion: kagent.dev/v1alpha2
kind: ModelConfig
metadata:
  name: llama3-model-config
  namespace: kagent
spec:
  model: llama3
  provider: Ollama
  ollama:
    host: http://ollama.ollama.svc.cluster.local:80
EOF
```

6. You should be able to see the Model connected
```
kubectl get modelconfig -n kagent
NAME                   PROVIDER    MODEL
default-model-config   Anthropic   claude-3-5-haiku-20241022
llama3-model-config    Ollama      llama3
```

7. If you go into kagent, you'll now see Llama as an option.

![](images/llama.png)

8. Create a new Agent in kagent with Llama being the Model.

9. Once created, check to confirm that the Agent is in a ready status.
```
kubectl get agent -n kagent
```

You can now create a new Agent with Llama3 as the Model

Once you do, you can test it out. Please note that to get a response, it might be a little slow depending on how large your Worker Nodes are.

![](images/date.png)

## Agentic Monitoring and Observability

### Kube-Prometheus Install

1. Install the Helm Chart
```
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

helm upgrade --install kube-prometheus-stack prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace \
  --set prometheus.prometheusSpec.serviceMonitorSelectorNilUsesHelmValues=false \
  --set prometheus.prometheusSpec.podMonitorSelectorNilUsesHelmValues=false \
  --set prometheus.prometheusSpec.ruleSelectorNilUsesHelmValues=false \
  --set prometheus.prometheusSpec.retention=30d \
  --set prometheus.prometheusSpec.storageSpec.volumeClaimTemplate.spec.resources.requests.storage=50Gi
```

2. Access Grafana
```
kubectl port-forward -n monitoring svc/kube-prometheus-stack-grafana 3000:80
```

- Username: admin
- Password: `kubectl get secret kube-prometheus-stack-grafana -n monitoring -o jsonpath='{.data.admin-password}' | base64 --decode`


### Agentic Metrics

1. Deploy a Pod Monitor

The `PodMonitor` selector matches `gateway.networking.k8s.io/gateway-class-name: agentgateway`, which will match any Gateway that uses the agentgateway GatewayClass
```
kubectl apply -f - <<EOF
apiVersion: monitoring.coreos.com/v1
kind: PodMonitor
metadata:
  name: agentgateway
  namespace: monitoring
  labels:
    app: agentgateway
    release: kube-prometheus
spec:
  selector:
    matchLabels:
      gateway.networking.k8s.io/gateway-class-name: agentgateway-enterprise
  podMetricsEndpoints:
  - port: metrics
    path: /metrics
    interval: 30s
    scrapeTimeout: 10s
EOF
```

2. Check the dashboard for metrics under the **Explore** tab