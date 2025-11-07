# k8s-ai-agents-demo

Reference Kubernetes demos for AI agents and RAG patterns. Shows how to wire agents, vector stores, observability, and secure images using the AI Governance Lab stack.

## What’s included
- Agent services (chat/tools) runnable in‑cluster
- Optional RAG with Qdrant
- Minimal FastAPI serving patterns
- Hooks for MLflow tracking and Evidently monitoring
- Helm/K8s manifests and scripts to deploy locally or to any K8s

## Prerequisites (Windows)
- A Kubernetes cluster (Rancher Desktop, Minikube, or kind)
- Docker/Containerd runtime
- kubectl 1.27+ and Helm 3.12+
- Git; optional: Make, PowerShell 7+
- Optionally build base images from docker_images_for_ai_agents

Verify:
```powershell
kubectl version --short
helm version
kubectl get nodes
```

## Quick start
1) Clone and set context
```powershell
git clone https://github.com/AI-Governance-Lab/AI-Governance-Lab.git
cd AI-Governance-Lab\k8s-ai-agents-demo
kubectl create ns ai-agents-demo
```

2) (Optional) Deploy Qdrant for RAG
```powershell
helm repo add qdrant https://qdrant.github.io/helm-charts
helm repo update
helm install qdrant qdrant/qdrant -n ai-agents-demo
kubectl wait --for=condition=Available deploy/qdrant -n ai-agents-demo --timeout=120s
```

3) Build or choose images
- Prefer hardened bases from docker_images_for_ai_agents (e.g., serving-fastapi, rag-runtime).
- Push to a registry reachable by the cluster or load into kind if using kind.

Example (adjust paths/tags):
```powershell
# From repo root
cd docker_images_for_ai_agents
docker build -f serving-fastapi/Dockerfile -t local/serving-fastapi:py3.11-v1 .
```

4) Deploy demo agents
- If using Helm charts in this repo, set image and config in values.yaml, then:
```powershell
# Adjust chart path if different
helm upgrade --install agents ./charts/agents -n ai-agents-demo `
  --set image.repository=local/serving-fastapi `
  --set image.tag=py3.11-v1
```

- If using raw manifests:
```powershell
kubectl apply -n ai-agents-demo -f manifests/
```

5) Access the service
```powershell
kubectl get svc -n ai-agents-demo
kubectl port-forward -n ai-agents-demo svc/agents 8080:80
# Then browse http://localhost:8080
```

## Configuration
Common settings (via values.yaml or env):
- MODEL_BACKEND: ollama|openai|hf
- QDRANT_URL: http://qdrant.ai-agents-demo.svc.cluster.local:6333
- TRACKING_URI: MLflow server URI (optional)
- LOG_LEVEL: info|debug

Example Helm overrides:
```yaml
image:
  repository: local/serving-fastapi
  tag: py3.11-v1
env:
  - name: LOG_LEVEL
    value: info
  - name: QDRANT_URL
    value: http://qdrant.ai-agents-demo.svc.cluster.local:6333
resources:
  limits:
    cpu: "1"
    memory: "1Gi"
```

## Security and governance
- Run as non‑root; drop capabilities; read‑only root FS where possible
- Pin dependencies; generate SBOM (syft) and scan (trivy) for images
- Use Kubernetes RBAC and least‑privilege service accounts
- Optionally add policy with OPA/Kyverno and image signing (cosign)

## Troubleshooting
- Image pull errors: ensure registry access or load images into the cluster
- CrashLoopBackOff: check env vars and secret mounts
- Service not reachable: verify Service/Ingress and port-forward
```powershell
kubectl describe pod -n ai-agents-demo -l app=agents
kubectl logs -n ai-agents-demo -l app=agents --tail=100
```

## Cleanup
```powershell
helm uninstall agents -n ai-agents-demo
helm uninstall qdrant -n ai-agents-demo
kubectl delete ns ai-agents-demo
```

## License
Follows the parent repository license. Review per‑component licenses as needed.