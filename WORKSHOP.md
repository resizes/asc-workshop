# Workshop: LLMOps Platform - Governing AI in Difficult Times

## Description

Nowadays almost everyone uses LLMs in their projects. But what would happen if major providers decided to multiply their service costs by 10?

In this workshop we will deploy an LLM inference server on a local Kubernetes cluster using **kind** and **vLLM**, demonstrating how we can achieve technological sovereignty over our AI infrastructure.

---

## Prerequisites

### Required Software

| Tool | Minimum Version | Description |
|------|----------------|-------------|
| Docker | 20.10+ | Container runtime |
| kubectl | 1.28+ | Kubernetes CLI |
| kind | 0.20+ | Kubernetes in Docker |
| curl | - | For making HTTP requests |

### System Requirements

- **RAM**: Minimum 8 GB free (16 GB recommended)
- **Disk**: Minimum 20 GB free (the vLLM image and model take up space)
- **CPU**: Minimum 4 cores
- **OS**: macOS, Linux or Windows with WSL2

### Hugging Face Account

You will need a Hugging Face token to download the model. If you don't have an account:

1. Sign up at https://huggingface.co/join
2. Go to https://huggingface.co/settings/tokens
3. Create a token with read permissions
4. Accept the model license at https://huggingface.co/meta-llama/Llama-3.2-1B-Instruct

---

## Step 0: Install the Tools

### Install Docker

If you don't have Docker installed yet:

**macOS:**
```bash
brew install --cask docker
```

**Linux (Ubuntu/Debian):**
```bash
sudo apt-get update
sudo apt-get install -y docker.io
sudo usermod -aG docker $USER
# Restart the session for the group to take effect
```

Verify Docker works:
```bash
docker version
```

### Install kubectl

**macOS:**
```bash
brew install kubectl
```

**Linux:**
```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/
```

Verify:
```bash
kubectl version --client
```

### Install kind

**macOS:**
```bash
brew install kind
```

**Linux:**
```bash
[ $(uname -m) = x86_64 ] && curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.27.0/kind-linux-amd64
[ $(uname -m) = aarch64 ] && curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.27.0/kind-linux-arm64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind
```

Verify:
```bash
kind version
```

---

## Step 1: Create the Kubernetes Cluster

We will use a kind configuration with 1 control-plane node and 2 workers:

```bash
kind create cluster --config kind-config.yaml
```

Verify the cluster is running:
```bash
kubectl cluster-info
kubectl get nodes
```

You should see something like:
```
NAME                   STATUS   ROLES           AGE   VERSION
llmops-control-plane   Ready    control-plane   1m    v1.35.0
llmops-worker          Ready    <none>          1m    v1.35.0
llmops-worker2         Ready    <none>          1m    v1.35.0
```

---

## Step 2: Create the Namespace

We will work in a dedicated namespace:

```bash
kubectl create namespace llmops
```

---

## Step 3: Create the Secret with the Hugging Face Token

Replace `YOUR_TOKEN_HERE` with your real Hugging Face token:

```bash
kubectl create secret generic hf-token-secret \
  --namespace llmops \
  --from-literal=token=YOUR_TOKEN_HERE
```

Verify it was created:
```bash
kubectl get secret hf-token-secret -n llmops
```

---

## Step 4: Create the PersistentVolumeClaim

This will reserve space to store the downloaded model:

```bash
kubectl apply -n llmops -f pvc.yaml
```

Contents of `pvc.yaml`:
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: vllm-models
spec:
  accessModes:
    - ReadWriteOnce
  volumeMode: Filesystem
  resources:
    requests:
      storage: 10Gi
```

Verify:
```bash
kubectl get pvc -n llmops
```

---

## Step 5: Deploy vLLM

Now we deploy the vLLM inference server with the Llama 3.2 1B Instruct model (a small model that can run on CPU):

```bash
kubectl apply -n llmops -f deployment.yaml
```

Contents of `deployment.yaml`:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: vllm-server
  labels:
    app.kubernetes.io/name: vllm
spec:
  replicas: 1
  selector:
    matchLabels:
      app.kubernetes.io/name: vllm
  template:
    metadata:
      labels:
        app.kubernetes.io/name: vllm
    spec:
      containers:
      - name: vllm
        image: public.ecr.aws/q9t5s3a7/vllm-arm64-cpu-release-repo:latest
        command: ["/bin/sh", "-c"]
        args:
        - "vllm serve meta-llama/Llama-3.2-1B-Instruct --dtype float32 --max-model-len 2048 --gpu-memory-utilization 0.4"
        env:
        - name: HF_TOKEN
          valueFrom:
            secretKeyRef:
              name: hf-token-secret
              key: token
        ports:
        - containerPort: 8000
        resources:
          requests:
            cpu: "6"
            memory: 16Gi
          limits:
            cpu: "8"
            memory: 24Gi
        volumeMounts:
        - name: model-storage
          mountPath: /root/.cache/huggingface
        livenessProbe:
          httpGet:
            path: /health
            port: 8000
          initialDelaySeconds: 120
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /health
            port: 8000
          initialDelaySeconds: 120
          periodSeconds: 5
      volumes:
      - name: model-storage
        persistentVolumeClaim:
          claimName: vllm-models
```

> **Note**: The first time it will take several minutes because it has to download the model (~2.5 GB). Subsequent runs will be faster thanks to the PVC.

---

## Step 6: Create the Service

Expose the vLLM server inside the cluster:

```bash
kubectl apply -n llmops -f service.yaml
```

Contents of `service.yaml`:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: vllm-server
spec:
  selector:
    app.kubernetes.io/name: vllm
  ports:
  - protocol: TCP
    port: 8000
    targetPort: 8000
  type: ClusterIP
```

---

## Step 7: Verify the Deployment

Monitor the pod status:
```bash
kubectl get pods -n llmops -w
```

Wait until the pod is in `Running` state and `READY 1/1`.

Check the logs to see the model download progress:
```bash
kubectl logs -n llmops -l app.kubernetes.io/name=vllm -f
```

When you see a message like `Uvicorn running on http://0.0.0.0:8000`, the server is ready.

---

## Step 8: Test the Model

Open a port-forward to access the service from your local machine:

```bash
kubectl port-forward -n llmops svc/vllm-server 8000:8000
```

In another terminal, make a test request:

### List available models (OpenAI API compatible)

```bash
curl http://localhost:8000/v1/models | jq .
```

### Make a completion

```bash
curl http://localhost:8000/v1/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "meta-llama/Llama-3.2-1B-Instruct",
    "prompt": "Kubernetes is a container orchestration platform that",
    "max_tokens": 50,
    "temperature": 0.7
  }' | jq .
```

### Make a chat completion

```bash
curl http://localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "meta-llama/Llama-3.2-1B-Instruct",
    "messages": [
      {"role": "system", "content": "You are an expert DevOps assistant."},
      {"role": "user", "content": "What is LLMOps and why is it important?"}
    ],
    "max_tokens": 200,
    "temperature": 0.7
  }' | jq .
```

---

## Step 9: Use with an OpenAI Client (optional)

vLLM exposes an OpenAI-compatible API, so you can use any OpenAI SDK pointing to your local server:

```python
from openai import OpenAI

client = OpenAI(
    base_url="http://localhost:8000/v1",
    api_key="not-required",
)

response = client.chat.completions.create(
    model="meta-llama/Llama-3.2-1B-Instruct",
    messages=[
        {"role": "system", "content": "You are an expert DevOps assistant."},
        {"role": "user", "content": "Explain what technological sovereignty in AI means."},
    ],
    max_tokens=300,
)

print(response.choices[0].message.content)
```

---

## Step 10: Code Generation with the Model

One of the most practical uses of a self-hosted LLM is code generation. Since Llama 3.2 Instruct is a capable coding model, we can use it to write, explain, and debug code — all running locally.

### Generate a Python function

Ask the model to write a function for you:

```bash
curl http://localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "meta-llama/Llama-3.2-1B-Instruct",
    "messages": [
      {"role": "system", "content": "You are an expert Python developer. Respond only with clean, working code and a brief explanation."},
      {"role": "user", "content": "Write a Python function that retries a failed HTTP request up to 3 times with exponential backoff."}
    ],
    "max_tokens": 400,
    "temperature": 0.2
  }' | jq -r '.choices[0].message.content'
```

> **Note**: We use `temperature: 0.2` for code — lower temperature means more deterministic, less creative output, which is what you want for code generation.

### Generate a Kubernetes manifest

Ask it to generate infrastructure code:

```bash
curl http://localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "meta-llama/Llama-3.2-1B-Instruct",
    "messages": [
      {"role": "system", "content": "You are a Kubernetes expert. Output only valid YAML."},
      {"role": "user", "content": "Write a Kubernetes CronJob that runs a curl command to a health endpoint every 5 minutes and logs the response."}
    ],
    "max_tokens": 400,
    "temperature": 0.1
  }' | jq -r '.choices[0].message.content'
```

### Code review

Ask it to review and improve existing code:

```bash
curl http://localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "meta-llama/Llama-3.2-1B-Instruct",
    "messages": [
      {"role": "system", "content": "You are a senior software engineer doing a code review. Be concise and focus on bugs and improvements."},
      {"role": "user", "content": "Review this Python code:\n\ndef get_user(id):\n    db = connect_db()\n    result = db.query(f'\''SELECT * FROM users WHERE id = {id}'\'')\n    return result"}
    ],
    "max_tokens": 300,
    "temperature": 0.3
  }' | jq -r '.choices[0].message.content'
```

---

## Step 11: Reasoning with Step-by-Step Prompting

Small models like Llama 3.2 1B don't have built-in chain-of-thought reasoning like larger models (e.g. DeepSeek-R1, QwQ), but you can elicit structured reasoning by prompting them explicitly. This is a key LLMOps technique: getting more out of a small, cheap model through prompt engineering.

### Chain-of-thought prompting

Ask the model to reason step by step before giving an answer:

```bash
curl http://localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "meta-llama/Llama-3.2-1B-Instruct",
    "messages": [
      {"role": "system", "content": "You are a debugging expert. Think through the problem step by step before giving your answer. Format your response as:\n\nThinking:\n<your reasoning>\n\nAnswer:\n<your conclusion>"},
      {"role": "user", "content": "A Kubernetes pod keeps restarting every 2 minutes. The logs show the process exits with code 137. What is causing this and how do I fix it?"}
    ],
    "max_tokens": 500,
    "temperature": 0.3
  }' | jq -r '.choices[0].message.content'
```

### Multi-step problem decomposition

Break a complex problem into steps:

```bash
curl http://localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "meta-llama/Llama-3.2-1B-Instruct",
    "messages": [
      {"role": "system", "content": "You are a senior DevOps architect. When given a problem, first list the steps to solve it, then implement each step."},
      {"role": "user", "content": "I need to set up a CI/CD pipeline that: builds a Docker image, runs tests, pushes to a registry, and deploys to Kubernetes. Give me the GitHub Actions workflow."}
    ],
    "max_tokens": 600,
    "temperature": 0.2
  }' | jq -r '.choices[0].message.content'
```

### Python client with reasoning loop

A more advanced example: a Python script that uses the model in a reasoning loop, feeding its own output back as context:

```python
from openai import OpenAI

client = OpenAI(
    base_url="http://localhost:8000/v1",
    api_key="not-required",
)

def reason_and_code(problem: str) -> dict:
    # Step 1: reason about the problem
    reasoning = client.chat.completions.create(
        model="meta-llama/Llama-3.2-1B-Instruct",
        messages=[
            {"role": "system", "content": "You are a software architect. Analyze the problem and list the key requirements and edge cases. Be concise."},
            {"role": "user", "content": problem},
        ],
        max_tokens=300,
        temperature=0.3,
    )
    analysis = reasoning.choices[0].message.content

    # Step 2: generate code informed by the analysis
    implementation = client.chat.completions.create(
        model="meta-llama/Llama-3.2-1B-Instruct",
        messages=[
            {"role": "system", "content": "You are an expert Python developer. Write clean, production-ready code with error handling."},
            {"role": "user", "content": problem},
            {"role": "assistant", "content": f"Analysis:\n{analysis}"},
            {"role": "user", "content": "Now implement the solution in Python based on this analysis."},
        ],
        max_tokens=500,
        temperature=0.1,
    )
    code = implementation.choices[0].message.content

    return {"analysis": analysis, "code": code}


result = reason_and_code(
    "Write a function that reads a Kubernetes deployment YAML file "
    "and returns the list of container images used."
)

print("=== ANALYSIS ===")
print(result["analysis"])
print("\n=== IMPLEMENTATION ===")
print(result["code"])
```

> **Key insight**: This two-step pattern — reason first, then generate — significantly improves output quality from small models. It's the same principle behind chain-of-thought prompting used in production LLMOps pipelines.

---

## Step 12: Deploy a Dedicated Coding Model

The Llama 3.2 1B model is a great general-purpose model, but for coding tasks a specialized model does significantly better. We will deploy **Qwen 2.5 Coder 1.5B Instruct** as a second independent deployment — no Hugging Face token required.

This demonstrates a key LLMOps pattern: **running multiple specialized models in the same cluster**, routing different workloads to the right model.

### Deploy the PVC, Deployment and Service

```bash
kubectl apply -n llmops -f pvc-coder.yaml
kubectl apply -n llmops -f deployment-coder.yaml
kubectl apply -n llmops -f service-coder.yaml
```

Verify both deployments are running:

```bash
kubectl get pods -n llmops
```

You should see both pods:
```
NAME                           READY   STATUS    RESTARTS   AGE
vllm-server-xxx                1/1     Running   0          10m
vllm-coder-xxx                 1/1     Running   0          2m
```

### Port-forward the coder model

Open a second port-forward on a different local port so both models are accessible simultaneously:

```bash
kubectl port-forward -n llmops svc/vllm-coder 8001:8000
```

Verify the model is available:

```bash
curl http://localhost:8001/v1/models | jq .
```

---

## Step 13: Coding Examples with the Specialized Model

All requests go to port `8001` (the coder model). Compare the quality of responses with the general model on port `8000`.

### Generate a Python function with tests

```bash
curl http://localhost:8001/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "Qwen/Qwen2.5-Coder-1.5B-Instruct",
    "messages": [
      {"role": "system", "content": "You are an expert Python developer. Write clean, well-tested code."},
      {"role": "user", "content": "Write a Python function that parses a Kubernetes resource string like '\''100m'\'' (millicores) or '\''2'\'' (cores) and returns the value in millicores as an integer. Include unit tests using pytest."}
    ],
    "max_tokens": 600,
    "temperature": 0.1
  }' | jq -r '.choices[0].message.content'
```

### Generate a Helm values file

```bash
curl http://localhost:8001/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "Qwen/Qwen2.5-Coder-1.5B-Instruct",
    "messages": [
      {"role": "system", "content": "You are a Kubernetes and Helm expert. Output only valid YAML with comments."},
      {"role": "user", "content": "Write a Helm values.yaml for a web application deployment with: configurable replicas, resource requests/limits, ingress with TLS, horizontal pod autoscaler, and a PostgreSQL dependency."}
    ],
    "max_tokens": 700,
    "temperature": 0.1
  }' | jq -r '.choices[0].message.content'
```

### Debug code

```bash
curl http://localhost:8001/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "Qwen/Qwen2.5-Coder-1.5B-Instruct",
    "messages": [
      {"role": "system", "content": "You are a debugging expert. Identify all bugs, explain each one, and provide the fixed code."},
      {"role": "user", "content": "Find the bugs in this Python code:\n\nimport threading\n\nresults = []\n\ndef fetch(url):\n    import urllib.request\n    data = urllib.request.urlopen(url).read()\n    results.append(data)\n\nurls = ['\''http://example.com'\''] * 10\nthreads = [threading.Thread(target=fetch, args=(u,)) for u in urls]\nfor t in threads: t.start()\nprint(f'\''Fetched {len(results)} pages'\'')"}
    ],
    "max_tokens": 500,
    "temperature": 0.2
  }' | jq -r '.choices[0].message.content'
```

### Compare both models side by side

This Python script sends the same coding prompt to both models and prints the responses for comparison:

```python
from openai import OpenAI

general = OpenAI(base_url="http://localhost:8000/v1", api_key="not-required")
coder = OpenAI(base_url="http://localhost:8001/v1", api_key="not-required")

prompt = "Write a Python context manager that measures and prints the execution time of a code block."

def ask(client, model, label):
    response = client.chat.completions.create(
        model=model,
        messages=[
            {"role": "system", "content": "You are an expert Python developer. Be concise."},
            {"role": "user", "content": prompt},
        ],
        max_tokens=400,
        temperature=0.1,
    )
    print(f"\n{'='*60}")
    print(f"Model: {label}")
    print('='*60)
    print(response.choices[0].message.content)

ask(general, "meta-llama/Llama-3.2-1B-Instruct", "Llama 3.2 1B (general)")
ask(coder, "Qwen/Qwen2.5-Coder-1.5B-Instruct", "Qwen 2.5 Coder 1.5B (specialized)")
```

> **Takeaway**: The specialized coder model typically produces more idiomatic code, better error handling, and more complete solutions — even at a similar parameter count. Choosing the right model for the workload is a core LLMOps decision.

---

## Uninstall

### Delete the Kubernetes resources

```bash
kubectl delete namespace llmops
```

This will delete all resources in the namespace (both deployments, services, PVCs, and the secret).

### Delete the kind cluster

```bash
kind delete cluster --name llmops
```

### Verify everything was cleaned up

```bash
kind get clusters
docker ps
```

No kind containers should remain running.

### (Optional) Clean up Docker images

If you want to reclaim disk space:

```bash
docker system prune -a
```

> **Warning**: This removes ALL unused Docker images, containers, and volumes.

---

## Troubleshooting

### The pod stays in Pending

```bash
kubectl describe pod -n llmops -l app.kubernetes.io/name=vllm
```

Common causes:
- Not enough CPU/memory on the nodes. Reduce the `requests` in the deployment.
- The PVC cannot be provisioned. Check with `kubectl get pvc -n llmops`.

### The pod keeps restarting (CrashLoopBackOff)

```bash
kubectl logs -n llmops -l app.kubernetes.io/name=vllm --previous
```

Common causes:
- Invalid Hugging Face token or insufficient permissions for the model.
- OOM (Out of Memory). Increase the memory `limits` or use a smaller model.

### The model takes too long to respond

On CPU, inference is significantly slower than on GPU. It is normal for a 50-token response to take 30-60 seconds. This is part of the demonstration: understanding the trade-offs between cost (GPU) and latency.

### Probe errors (liveness/readiness)

If the pod is killed before it is ready, increase `initialDelaySeconds` in the probes to give the model time to load. On Apple Silicon with Rosetta (x86_64 emulation), loading can take 5-10 minutes — set `initialDelaySeconds: 600`.

---

## Additional Resources

- [vLLM Documentation](https://docs.vllm.ai/)
- [vLLM on Kubernetes](https://docs.vllm.ai/en/stable/deployment/k8s/)
- [kind - Kubernetes in Docker](https://kind.sigs.k8s.io/)
- [Models supported by vLLM](https://docs.vllm.ai/en/stable/models/supported_models.html)
- [OpenAI API compatibility](https://docs.vllm.ai/en/stable/serving/openai_compatible_server.html)
