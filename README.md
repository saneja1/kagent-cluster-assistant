# Natural-language Kubernetes troubleshooting with kagent

Ask a cluster questions in plain English. A **supervisor** agent routes each request to specialists that **diagnose** (read-only) or **fix** (patch/apply) workloads on a local kind cluster.

This repo is **my multi-agent design and demos** built **on top of** open-source [kagent](https://kagent.dev). It does not reimplement the kagent platform.

## What I built

| Piece | Role |
|--------|------|
| `supervisor-agent` | Single front door: routes intents, can chain diagnose → fix |
| `diagnose-agent` | Read-only investigation (pods, events, logs, YAML) |
| `fixer-agent` | Applies small fixes (patch/apply/create) and verifies |
| `landing/` | Simple front page that opens supervisor chat |

Stock kagent demo agents (Helm, Istio, Cilium, …) can also be reached via the supervisor when relevant.

## Architecture

```text
You → landing page → kagent UI → supervisor-agent
                                      ├─ diagnose-agent  (read tools)
                                      ├─ fixer-agent     (read + write tools)
                                      └─ other specialists (optional)
                                              ↓
                                         kind cluster (demo apps)
```

**Design choice:** separate diagnose vs fix for least privilege. The supervisor may call more than one agent in a turn (e.g. list unhealthy pods, then fix).

## Repo layout

```text
landing/                 # Front page (“Chat with me” → supervisor)
agents/                  # Exported Agent YAMLs (my custom agents)
README.md
```

## Prerequisites

- Docker + [kind](https://kind.sigs.k8s.io/)
- `kubectl`, Helm 3
- [kagent CLI](https://kagent.dev) **v0.9.x** (matches this lab)
- An LLM API key (this lab used **Anthropic**)

## Quick start

### 1. Create a kind cluster

```bash
kind create cluster --name kagent-lab
```

### 2. Install kagent (example: Anthropic)

```bash
export ANTHROPIC_API_KEY='…'

helm install kagent-crds oci://ghcr.io/kagent-dev/kagent/helm/kagent-crds \
  --version 0.9.9 \
  --namespace kagent \
  --create-namespace

helm install kagent oci://ghcr.io/kagent-dev/kagent/helm/kagent \
  --version 0.9.9 \
  --namespace kagent \
  --set providers.default=anthropic \
  --set providers.anthropic.apiKey=$ANTHROPIC_API_KEY
```

Wait until pods in `kagent` are Ready.

### 3. Apply my custom agents

```bash
kubectl apply -f agents/diagnose-agent.yaml
kubectl apply -f agents/fixer-agent.yaml
kubectl apply -f agents/supervisor-agent.yaml
```

### 4. Demo apps (optional)

```bash
kubectl create namespace demo
kubectl -n demo create deployment hello --image=nginx:1.27
kubectl -n demo create deployment cache --image=redis:7
```

### 5. Open UI + landing page

```bash
kubectl -n kagent port-forward service/kagent-ui 8089:8080
```

In another terminal:

```bash
cd landing && python3 -m http.server 5500 --bind 0.0.0.0
```

- Landing: http://localhost:5500  
- Supervisor chat (direct): http://localhost:8089/agents/kagent/supervisor-agent/chat  

Edit `landing/config.js` if your supervisor URL differs.

## Demo script

Break a deployment:

```bash
kubectl -n demo set image deployment/hello nginx=nginx:wrong-image-xyz
```

Ask **supervisor**:

1. Diagnose only — expect `Routed to: diagnose-agent`  
   > What’s wrong with hello in demo? Don’t change anything.

2. Fix — expect `Routed to: fixer-agent` (or diagnose then fixer)  
   > Fix hello in demo with image nginx:1.27. Do not add extra containers.

3. Chain — expect both  
   > Check pods in demo. If unhealthy, diagnose, then fix image issues with nginx:1.27.

Confirm in the terminal:

```bash
kubectl -n demo get pods
```

## Portfolio highlights

- Multi-agent routing and optional **chaining** (diagnose → fix)
- Clear **read vs write** tool split for safer automation
- Reproducible failure demos on kind (`ImagePullBackOff`)
- Lightweight landing page as the product front door

## Credits

- Platform: [kagent](https://github.com/kagent-dev/kagent) (Apache-2.0)
- Local cluster: kind
- Model provider used in this lab: Anthropic Claude

## License

My agents, landing page, and docs in this repo: add your preferred license when you publish.
