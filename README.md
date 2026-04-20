# llm-k8s-deploy

Deploy an open source LLM on Kubernetes with Grafana monitoring and a chatbot UI. Runs entirely on your local machine. No cloud required.

## What this is

A complete local stack for running a language model inside a Kubernetes cluster. Includes cluster setup, model deployment, Prometheus and Grafana monitoring, and a browser-based chatbot frontend.

## Stack

| Component | Tool |
|-----------|------|
| Kubernetes | Minikube |
| LLM Runtime | Ollama |
| Model | TinyLlama 1.1B (GGUF) |
| Monitoring | Prometheus + Grafana via kube-prometheus-stack |
| Chatbot UI | Vanilla HTML/JS |

## Requirements

- Ubuntu 20.04 or later (tested on 24.04)
- Docker installed and running
- At least 8GB RAM (16GB recommended)
- At least 20GB free disk space
- kubectl, minikube, helm installed

## Quick Start

### 1. Start the cluster

```bash
minikube start --driver=docker --cpus=4 --memory=6144
kubectl get nodes
```

### 2. Install Ollama and pull a model

```bash
sudo snap install ollama
wget "https://huggingface.co/TheBloke/TinyLlama-1.1B-Chat-v1.0-GGUF/resolve/main/tinyllama-1.1b-chat-v1.0.Q4_K_M.gguf" -O ~/tinyllama.gguf

cat > ~/Modelfile << 'EOF'
FROM /home/$USER/tinyllama.gguf
SYSTEM "You are a helpful AI assistant."
EOF

ollama create tinyllama -f ~/Modelfile
OLLAMA_ORIGINS="*" ollama serve &
```

### 3. Deploy Ollama to Kubernetes

```bash
docker pull ollama/ollama:latest
minikube image load ollama/ollama:latest
kubectl apply -f k8s/ollama-deployment.yaml
kubectl get pods -w
```

### 4. Set up monitoring

Pull and load all required images first (required for restricted network environments):

```bash
docker pull grafana/grafana:latest
docker pull quay.io/prometheus/prometheus:v3.11.2
docker pull quay.io/prometheus/node-exporter:v1.11.1
docker pull quay.io/prometheus/alertmanager:v0.32.0
docker pull quay.io/prometheus-operator/prometheus-operator:v0.82.0
docker pull quay.io/prometheus-operator/prometheus-config-reloader:v0.90.1
docker pull quay.io/kiwigrid/k8s-sidecar:2.6.0
docker pull registry.k8s.io/kube-state-metrics/kube-state-metrics:v2.18.0

for image in \
  grafana/grafana:latest \
  quay.io/prometheus/prometheus:v3.11.2 \
  quay.io/prometheus/node-exporter:v1.11.1 \
  quay.io/prometheus/alertmanager:v0.32.0 \
  quay.io/prometheus-operator/prometheus-operator:v0.82.0 \
  quay.io/prometheus-operator/prometheus-config-reloader:v0.90.1 \
  quay.io/kiwigrid/k8s-sidecar:2.6.0 \
  registry.k8s.io/kube-state-metrics/kube-state-metrics:v2.18.0; do
  minikube image load $image
done
```

Install the Helm chart:

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts --insecure-skip-tls-verify
helm repo update --insecure-skip-tls-verify
kubectl create namespace monitoring

helm install monitoring prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --insecure-skip-tls-verify \
  --timeout 10m \
  --set grafana.adminPassword=admin123 \
  --set prometheusOperator.admissionWebhooks.enabled=false \
  --set prometheusOperator.admissionWebhooks.patch.enabled=false \
  --set prometheusOperator.tls.enabled=false

kubectl apply -f k8s/grafana-standalone.yaml
```

Access Grafana:

```bash
kubectl port-forward svc/grafana-standalone 3000:3000 -n monitoring
```

Open http://localhost:3000 and login with admin / admin123.

Add Prometheus as a data source using its ClusterIP:

```bash
kubectl get svc -n monitoring | grep prometheus
```

Use the ClusterIP shown for `monitoring-kube-prometheus-prometheus` with port 9090.

Import the Node Exporter Full dashboard by uploading the JSON file from `monitoring/node-exporter-dashboard.json`.

### 5. Run the chatbot

```bash
cd chatbot
python3 -m http.server 8080
```

Open http://localhost:8080 in your browser.

## Project Structure

```
llm-k8s-deploy/
├── k8s/
│   ├── ollama-deployment.yaml      Ollama deployment and service
│   └── grafana-standalone.yaml     Standalone Grafana deployment
├── chatbot/
│   └── index.html                  Beep chatbot UI
├── monitoring/
│   └── node-exporter-dashboard.json  Grafana dashboard JSON
└── README.md
```

## Troubleshooting

**Pods stuck in ImagePullBackOff**

Your cluster cannot reach external registries. Pull the image on your host and load it into Minikube manually:

```bash
docker pull <image-name>
minikube image load <image-name>
kubectl delete pod -l app=<app-name>
```

**Namespace stuck in Terminating**

Force remove the finalizers:

```bash
kubectl get namespace <name> -o json \
  | sed 's/"kubernetes"//' \
  | kubectl replace --raw /api/v1/namespaces/<name>/finalize -f -
```

**Chatbot shows connection error**

Make sure Ollama is running with CORS enabled:

```bash
pkill ollama
OLLAMA_ORIGINS="*" ollama serve &
```

And serve the chatbot through a local server, not as a file:// URL:

```bash
cd chatbot && python3 -m http.server 8080
```

**Grafana port-forward drops connection**

The pod may be restarting due to memory pressure. Check:

```bash
kubectl get pods -n monitoring | grep grafana
kubectl port-forward pod/<grafana-pod-name> 3000:3000 -n monitoring
```

## Using a better model

TinyLlama is small and fast but limited in quality. To use a better model, download Mistral 7B from HuggingFace and create it the same way:

```bash
wget "https://huggingface.co/TheBloke/Mistral-7B-Instruct-v0.1-GGUF/resolve/main/mistral-7b-instruct-v0.1.Q4_K_M.gguf" -O ~/mistral.gguf

cat > ~/MistralFile << 'EOF'
FROM /home/$USER/mistral.gguf
SYSTEM "You are a helpful AI assistant."
EOF

ollama create mistral -f ~/MistralFile
```

Then update the model name in `chatbot/index.html` from `tinyllama` to `mistral`.

## Blog post

Full write-up of the process including everything that went wrong: []

## Video

Walkthrough video: []
