# Vulnerable Overlay — Baseline Findings

**Scope:** the intentionally-insecure `vulnerable` overlay (`k8s/overlays/vulnerable`), deployed to
namespace `ai-lab-vuln`, with decoy targets in `ai-lab-decoy`.
**Date:** 2026-10-15.
**Status:** baseline only. These are the weaknesses *as built* — nothing here is fixed yet. The
hardened overlay addresses them later; this document is the "before" snapshot to compare against.

## How findings were gathered
- **Manual exploitation** inside the running lab, using the chat-api pod's mounted ServiceAccount
  token against the Kubernetes API.
- **Trivy image scan** — `trivy image chat-api:0.1.0` → [`attacks/trivy-image.txt`](../attacks/trivy-image.txt).
- **Trivy config scan** — `trivy config k8s/overlays/vulnerable` → [`attacks/trivy-config.txt`](../attacks/trivy-config.txt).

## Severity key
| Severity | Meaning in this lab |
|---|---|
| **Critical** | Exploited in the lab to reach secrets or cluster control, or a scanner Critical. |
| **High** | Exploited to cross a trust boundary, or a scanner High. |
| **Medium** | Real weakness, not directly exploited to impact on its own. |
| **Low** | Hardening gap / defence-in-depth; low standalone impact. |

A finding demonstrated by hand is rated by its proven impact, which can exceed the scanner's rating
for the same check (e.g. `create pods` is Medium to Trivy but was used here for cross-namespace
execution).

## Findings

### Exploited attack chain (manual)
| # | Vulnerability | How it was exploited / found | Severity | Evidence |
|---|---|---|---|---|
| 1 | ServiceAccount token auto-mounted in the chat-api pod and bound to a cluster-wide role (`automountServiceAccountToken: true` + the `chat-api-vuln` ClusterRoleBinding) | Read the token from the pod and used it as a bearer token to authenticate to the Kubernetes API as `system:serviceaccount:ai-lab-vuln:chat-api` | Critical | [`attacks/04-token-and-secret-read.txt`](../attacks/04-token-and-secret-read.txt) |
| 2 | Cluster-wide secret read (ClusterRole `get`/`list` on `secrets`, KSV-0041) | Direct API `GET` of Secret `db-credentials` in the **other** namespace `ai-lab-decoy` → `200 OK`, full secret returned | Critical | [`attacks/04-token-and-secret-read.txt`](../attacks/04-token-and-secret-read.txt) |
| 3 | Cluster-wide pod creation (ClusterRole `create` on `pods`, KSV-0048) | Same token created pod `attack-poc` in `ai-lab-decoy` → `201 Created`, then deleted. Proves arbitrary workload execution across namespaces | High | [`attacks/05-pod-create.txt`](../attacks/05-pod-create.txt) |
| 4 | Debug endpoint leaks all environment variables (`ENABLE_DEBUG=true`) | `GET /debug/env` returned every env var, including `API_KEY=FAKE-API-KEY`, with no authentication | High | [`attacks/06-debug-env.txt`](../attacks/06-debug-env.txt) |

### RBAC misconfigurations (Trivy config scan)
| # | Vulnerability | How it was exploited / found | Severity | Evidence |
|---|---|---|---|---|
| 5 | ClusterRole `chat-api-vuln` can manage `secrets` cluster-wide (KSV-0041) | Trivy config scan flagged `rbac.yaml` as **CRITICAL**; confirmed by finding #2 | Critical | [`attacks/trivy-config.txt`](../attacks/trivy-config.txt) |
| 6 | ClusterRole `chat-api-vuln` can `create` pods / workloads cluster-wide (KSV-0048) | Trivy config scan flagged `rbac.yaml` as MEDIUM; confirmed by finding #3 | High | [`attacks/trivy-config.txt`](../attacks/trivy-config.txt) |

### securityContext misconfigurations (Trivy config scan)
Each applies to **both** the `chat-api` and `ollama` containers.

| # | Vulnerability | How it was exploited / found | Severity | Evidence |
|---|---|---|---|---|
| 7 | Runs as root — `runAsUser: 0`, `runAsNonRoot` not set (KSV-0012, KSV-0118, KSV-0020/KSV-0105) | Trivy config scan (HIGH on KSV-0118 "default security context allows root privileges") | High | [`attacks/trivy-config.txt`](../attacks/trivy-config.txt) |
| 8 | Writable root filesystem — `readOnlyRootFilesystem: false` (KSV-0014) | Trivy config scan (HIGH) | High | [`attacks/trivy-config.txt`](../attacks/trivy-config.txt) |
| 9 | Privilege escalation allowed — `allowPrivilegeEscalation: true` (KSV-0001) | Trivy config scan (MEDIUM) | Medium | [`attacks/trivy-config.txt`](../attacks/trivy-config.txt) |
| 10 | No seccomp profile set (KSV-0030, KSV-0104) | Trivy config scan (MEDIUM/LOW) | Medium | [`attacks/trivy-config.txt`](../attacks/trivy-config.txt) |
| 11 | Linux capabilities not dropped — no `drop: ["ALL"]` (KSV-0003, KSV-0004, KSV-0106) | Trivy config scan (LOW) | Low | [`attacks/trivy-config.txt`](../attacks/trivy-config.txt) |

### Other config weaknesses (from the manifests)
| # | Vulnerability | How it was exploited / found | Severity | Evidence |
|---|---|---|---|---|
| 12 | API key stored in a ConfigMap instead of a Secret (KSV-01010) | Trivy config scan (MEDIUM) on `chat-api-config-patch.yaml`; the value also surfaced via finding #4 | Medium | [`attacks/trivy-config.txt`](../attacks/trivy-config.txt) |
| 13 | No resource requests/limits on either container → node-exhaustion DoS (KSV-0011, KSV-0015, KSV-0016, KSV-0018) | Trivy config scan (LOW); relevant to LLM prompt-flood DoS against Ollama | Low | [`attacks/trivy-config.txt`](../attacks/trivy-config.txt) |
| 14 | Services exposed as `NodePort`, including Ollama's unauthenticated API and chat-api's `/debug/env` | Read from `k8s/overlays/vulnerable/services-patch.yaml` (not flagged by the Trivy checks run) | Medium | [`services-patch.yaml`](../k8s/overlays/vulnerable/services-patch.yaml) |
| 15 | Namespace runs Pod Security Admission at `enforce: privileged` — no admission guardrails | Read from `k8s/overlays/vulnerable/namespace.yaml`; this is what lets finding #3's pod-create place a privileged pod | Medium | [`namespace.yaml`](../k8s/overlays/vulnerable/namespace.yaml) |
| 16 | No NetworkPolicy — any pod can reach chat-api, Ollama, and `ai-lab-decoy` | Absence confirmed across the overlay; enabled the cross-namespace reach in findings #2–3 | Medium | [`k8s/overlays/vulnerable/`](../k8s/overlays/vulnerable/) |

### Image vulnerabilities (Trivy image scan)
| # | Vulnerability | How it was exploited / found | Severity | Evidence |
|---|---|---|---|---|
| 17 | OS packages in the `python:3.13.16-slim` (Debian 13.7) base image: 166 CVEs (44 HIGH, 0 CRITICAL) | `trivy image chat-api:0.1.0` | High | [`attacks/trivy-image.txt`](../attacks/trivy-image.txt) |
| 18 | Python packages: 6 CVEs (4 HIGH) — e.g. urllib3 `CVE-2026-97687`, setuptools `CVE-2025-47273`, msgpack `GHSA-6v7p-g79w-8964` | `trivy image chat-api:0.1.0` | High | [`attacks/trivy-image.txt`](../attacks/trivy-image.txt) |

## Evidence index
| File | What it shows |
|---|---|
| `attacks/04-token-and-secret-read.txt` | Reading the pod's SA token and the cross-namespace `GET` of Secret `db-credentials` (200 OK). |
| `attacks/05-pod-create.txt` | Creating pod `attack-poc` in `ai-lab-decoy` (201 Created) and deleting it. |
| `attacks/06-debug-env.txt` | `GET /debug/env` response leaking `FAKE-API-KEY`. |
| `attacks/trivy-config.txt` | Full `trivy config` output for the vulnerable overlay. |
| `attacks/trivy-image.txt` | Full `trivy image` output for `chat-api:0.1.0`. |
| `attacks/07-hardened-verification.txt` | Live hardened-overlay checks: token absent, `/debug/env` 404, app healthy, decoy egress blocked. |

## Hardened Overlay — Verification
The hardened overlay (`k8s/overlays/hardened`) was deployed to namespace `ai-lab-hardened` and
verified on 2026-10-15. This section maps each baseline finding above to its closure status. Compare
the "blocked"/"not found" results here against the baseline evidence (e.g. finding #2's `200 OK`).

**Status key:**
- **Closed (verified live)** — re-tested on the running hardened deployment; evidence attached.
- **Closed (by design)** — the control is present in the manifests and the pods run healthy under PSA `restricted`; no separate live re-test.
- **Closed (by removal)** — the vulnerable construct no longer exists, so there is nothing left to test.
- **Open** — not addressed by this overlay.

| Finding | Status | How closed | Evidence |
|---|---|---|---|
| #1 SA token mounted | Closed (verified live) | `automountServiceAccountToken: false` on both SAs/pods; no RBAC to use it with | [`07`](../attacks/07-hardened-verification.txt) — `/var/run/secrets/...` absent (`ls` exit 2) |
| #2 cross-ns secret read | Closed (verified live) | default-deny egress + no token/RBAC | [`07`](../attacks/07-hardened-verification.txt) — `BLOCKED: timed out` (baseline: `200 OK`) |
| #3 cross-ns pod create | Closed (verified live) | default-deny egress + no RBAC | [`07`](../attacks/07-hardened-verification.txt) — `kubectl auth can-i create pods` → `no` for both hardened SAs (direct RBAC proof), plus egress `BLOCKED: timed out` (baseline: `201 Created`) |
| #4 `/debug/env` leak | Closed (verified live) | `ENABLE_DEBUG=false` | [`07`](../attacks/07-hardened-verification.txt) — `{"detail":"Not Found"}` |
| #5 RBAC get/list secrets | Closed (by removal) | ClusterRole/Binding removed entirely — no token exists to call the API, so no live re-test needed | [`k8s/overlays/hardened/`](../k8s/overlays/hardened/) |
| #6 RBAC create pods | Closed (by removal) | same — no cluster-scoped RBAC in the overlay | [`k8s/overlays/hardened/`](../k8s/overlays/hardened/) |
| #7–#11 securityContext (root, writable FS, priv-esc, caps, seccomp) | Closed (by design) | restricted `securityContext` on every container; confirmed by pods admitted and running under PSA `restricted` | design + [`07`](../attacks/07-hardened-verification.txt) (app healthy) |
| #12 key in ConfigMap | Closed (by design) | moved to a `Secret` (`chat-api-secret`) | [`k8s/overlays/hardened/chat-api-secret.yaml`](../k8s/overlays/hardened/chat-api-secret.yaml) |
| #13 no resource limits | Closed (by design) | requests/limits on chat-api and ollama | [`k8s/overlays/hardened/`](../k8s/overlays/hardened/) |
| #14 NodePort services | Closed (by design) | base Services are ClusterIP, left unpatched | [`k8s/base/`](../k8s/base/) |
| #15 PSA `privileged` | Closed (by design) | namespace enforces `restricted`; pods being admitted is the proof | design + [`07`](../attacks/07-hardened-verification.txt) |
| #16 no NetworkPolicy | Closed (verified live) | default-deny egress + scoped allows (DNS, chat-api→ollama, ollama→registry, pull-job→ollama) | [`07`](../attacks/07-hardened-verification.txt) — decoy egress `BLOCKED: timed out` |
| #17 OS image CVEs | **Open** | out of scope for an overlay; needs a base-image rebuild | [`attacks/trivy-image.txt`](../attacks/trivy-image.txt) |
| #18 Python package CVEs | **Open** | out of scope; needs dependency bumps | [`attacks/trivy-image.txt`](../attacks/trivy-image.txt) |

**No regression.** After hardening, `/healthz` returned `{"status":"ok"}` and `/chat` returned a normal
reply, so the controls didn't break the app — see [`attacks/07-hardened-verification.txt`](../attacks/07-hardened-verification.txt).

---
**Note on the Trivy `default` namespace.** `trivy config` scans the raw patch files in
`k8s/overlays/vulnerable/`, not the rendered overlay, so it reports objects in namespace `default`.
At deploy time Kustomize places them in `ai-lab-vuln` (and the RBAC objects are cluster-scoped). The
mismatch is a scanning artifact, not a second deployment.
