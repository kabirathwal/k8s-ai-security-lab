# k8s-ai-security-lab

A local Kubernetes lab (using [kind](https://kind.sigs.k8s.io/)) that runs a small LLM inference
service — [Ollama](https://ollama.com/) serving `qwen2.5:0.5b` behind a FastAPI `chat-api` — and
deploys it two ways for comparison: an **intentionally vulnerable** overlay and a **hardened** one,
built from a shared Kustomize base. The point is to demonstrate a realistic Kubernetes attack chain
against the vulnerable deployment and then show how specific, standard controls in the hardened
overlay close each step. Everything runs locally; all credentials in the repo are fake.

## Architecture

```
chat-api (FastAPI)  --HTTP-->  Ollama (qwen2.5:0.5b)      [the inference service, from k8s/base]
     |
     '-- POST /chat, GET /healthz, GET /debug/env

Deployed into one of two namespaces via Kustomize overlays:
  k8s/overlays/vulnerable  -> namespace ai-lab-vuln       (insecure on purpose)
  k8s/overlays/hardened    -> namespace ai-lab-hardened   (controls applied)

k8s/decoys -> namespace ai-lab-decoy                      (attack target, deployed separately)
  db-credentials Secret (fake) + a stand-in "postgres" Service/Deployment
```

- **chat-api** and **Ollama** are the application, defined once in `k8s/base/`.
- The **vulnerable** and **hardened** overlays patch that base differently; each deploys to its own
  namespace so both can run at the same time.
- The **decoy** namespace (`ai-lab-decoy`) is a separate target holding a fake database credential and
  a dummy service — what an attacker pivots to from a compromised `chat-api`.

## What's demonstrated

Against the **vulnerable** overlay, a four-step chain (full detail and evidence in
[`docs/findings.md`](docs/findings.md)):

1. **SA token theft** — the `chat-api` pod mounts its ServiceAccount token.
2. **Cross-namespace secret read** — that token, bound to a cluster-wide ClusterRole, reads the
   `db-credentials` Secret in `ai-lab-decoy` (`200 OK`).
3. **Cross-namespace pod creation** — the same token creates a pod in `ai-lab-decoy` (`201 Created`).
4. **Debug endpoint leak** — `GET /debug/env` returns all environment variables, including the API key.

The **hardened** overlay closes each one:

| Attack step | Control in the hardened overlay |
|---|---|
| SA token theft | `automountServiceAccountToken: false` on both ServiceAccounts/pods |
| Cross-namespace secret read | **No** ClusterRole/ClusterRoleBinding at all + default-deny-egress NetworkPolicy |
| Cross-namespace pod creation | same — confirmed by `kubectl auth can-i create pods` → `no` |
| Debug endpoint leak | `ENABLE_DEBUG=false`; API key moved to a `Secret` |
| (supporting) | restricted `securityContext`, resource limits, ClusterIP (not NodePort), namespace PSA `enforce: restricted` |

See [`docs/findings.md`](docs/findings.md) for the full baseline table (18 findings) and the
hardened-overlay verification section.

## Tech stack

kind · Calico CNI · Kustomize · Ollama · FastAPI · Trivy · kubectl

Pinned versions: `python:3.13.16-slim`, `ollama/ollama:0.40.4`, `nginx:1.30.6-alpine` (decoy),
Tigera/Calico operator `v1.42.6`; Python deps `fastapi==0.143.1`, `uvicorn==0.54.0`, `httpx==0.28.1`.

## How to reproduce

**Prerequisites:** `kind`, `kubectl`, and Docker.

**1. Create the cluster and install the CNI** (default CNI is disabled so Calico can enforce
NetworkPolicy):

```bash
kind create cluster --config cluster/kind-config.yaml        # cluster name: ai-lab
# Install the Tigera/Calico operator (this lab used operator v1.42.6), then the pinned CRs:
kubectl create -f cluster/custom-resources.yaml
kubectl get nodes          # wait until all 3 nodes are Ready
```

**2. Build the app image and load it into kind:**

```bash
docker build -t chat-api:0.1.0 app/
kind load docker-image chat-api:0.1.0 --name ai-lab
```

**3. Deploy the decoy target and an overlay:**

```bash
kubectl apply -k k8s/decoys                 # ai-lab-decoy (attack target)
kubectl apply -k k8s/overlays/vulnerable    # ai-lab-vuln
# and / or:
kubectl apply -k k8s/overlays/hardened      # ai-lab-hardened
```

The first `/chat` call is slow while Ollama pulls the model (a one-off Job downloads it onto a PVC).

**4. Verify** (condensed — full raw output is in `attacks/*.txt`):

```bash
# Vulnerable: debug endpoint leaks env (port-forward first)
kubectl -n ai-lab-vuln port-forward svc/chat-api 8080:80 &
curl -s localhost:8080/debug/env            # leaks API_KEY

# Hardened: the same checks now fail closed
kubectl -n ai-lab-hardened port-forward svc/chat-api 8081:80 &
curl -s localhost:8081/debug/env            # {"detail":"Not Found"}
kubectl -n ai-lab-hardened exec deploy/chat-api -- \
  ls /var/run/secrets/kubernetes.io/serviceaccount/     # No such file (no token mounted)
kubectl auth can-i create pods \
  --as=system:serviceaccount:ai-lab-hardened:chat-api -n ai-lab-decoy   # no
```

## Repo structure

```
app/                      FastAPI chat-api: main.py, Dockerfile, requirements.txt
cluster/                  kind cluster config + pinned Calico custom resources
k8s/base/                 shared, neutral manifests (chat-api + Ollama + model-pull Job)
k8s/overlays/vulnerable/  insecure patches; each weakness marked  # VULN:
k8s/overlays/hardened/    hardening patches; each control marked  # CONTROL:
k8s/decoys/               ai-lab-decoy: fake db-credentials Secret + stand-in service
attacks/                  evidence: trivy-*.txt scans, 04-07 attack/verification output
docs/findings.md          baseline findings table + hardened-overlay verification
```

## Status / scope

The overlay-based hardening addresses findings #1–#16 in `docs/findings.md`. **Image and dependency
CVEs (findings #17 and #18)** — OS packages in the base image and Python package CVEs reported by
Trivy — are a known open item and are **out of scope** for this overlay-based demonstration; closing
them requires rebuilding the image on a slimmer/updated base and bumping dependencies, not a Kustomize
patch.
