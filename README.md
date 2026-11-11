# Enterprise RBAC Chatbot — GitOps

Kubernetes deployment configuration for the [Enterprise RBAC Chatbot](https://github.com/shaikh4git/Enterprise-RBAC-Chatbot). This repository is the deployment source of truth: it contains the Kubernetes manifests, Kustomize configuration, Argo CD Application, and Argo Rollouts canary strategy.

- **Application source:** https://github.com/shaikh4git/Enterprise-RBAC-Chatbot
- **Deployment configuration:** https://github.com/shaikh4git/Enterprise-RBAC-Chatbot-GitOps

Argo CD watches this repository's `main` branch and reconciles the `app/` directory into the `enterprise-chatbot` namespace.

## Architecture

```text
Application repository
  shaikh4git/Enterprise-RBAC-Chatbot
          | build and publish image
          v
  shaikhdocker/enterprise-rbac-chatbot:<tag>
          | update tag in GitOps repository
          v
GitOps repository (main / app/)
          | Argo CD Application
          v
Kubernetes namespace: enterprise-chatbot
  +-- Rollout: enterprise-rbac-chatbot (2 replicas)
  |     +-- Streamlit container (8505)
  |     +-- ConfigMap: chatbot-config
  |     +-- Secret reference: chatbot-secrets
  |     +-- Argo Rollouts canary steps
  +-- Service: enterprise-rbac-chatbot (ClusterIP :8505)
  +-- Deployment: qdrant (1 replica)
        +-- Service: qdrant (ClusterIP :6333, :6334)
        +-- PVC: qdrant-pvc (2Gi)
```

The chatbot connects to Qdrant through `http://qdrant:6333`. The application image is built from the application repository; Kubernetes deployment changes are made here.

## Repository layout

```text
Enterprise-RBAC-Chatbot-GitOps/
├── argocd/
│   └── application.yaml       # Argo CD Application definition
└── app/
    ├── kustomization.yaml     # Kustomize entry point and image tag
    ├── namespace.yaml         # enterprise-chatbot namespace
    ├── configmap.yaml         # Non-sensitive application settings
    ├── rollout.yaml           # Chatbot Rollout and canary steps
    ├── chatbot-service.yaml   # Chatbot ClusterIP Service
    ├── qdrant-deployment.yaml # Qdrant Deployment
    ├── qdrant-service.yaml    # Qdrant ClusterIP Service
    └── qdrant-pvc.yaml        # Qdrant persistent volume claim
```

The files listed in `app/kustomization.yaml` are the resources Kustomize includes in the Argo CD application. The Secret is intentionally not stored here; it is referenced by the Rollout and must be created separately.

## Kubernetes resources

All application resources are deployed in `enterprise-chatbot`.

| Manifest | Kubernetes object | Purpose / important settings |
|---|---|---|
| `namespace.yaml` | Namespace | Creates `enterprise-chatbot`. |
| `configmap.yaml` | ConfigMap `chatbot-config` | Non-sensitive application configuration. |
| `rollout.yaml` | Argo Rollout `enterprise-rbac-chatbot` | Two chatbot replicas and canary steps. |
| `chatbot-service.yaml` | Service `enterprise-rbac-chatbot` | ClusterIP port 8505 → target port 8505. |
| `qdrant-deployment.yaml` | Deployment `qdrant` | Runs `qdrant/qdrant:v1.13.2`, one replica. |
| `qdrant-service.yaml` | Service `qdrant` | ClusterIP HTTP 6333 and gRPC 6334. |
| `qdrant-pvc.yaml` | PVC `qdrant-pvc` | Requests 2Gi, ReadWriteOnce. |

### Chatbot service

The `enterprise-rbac-chatbot` Service is ClusterIP-only, exposing port `8505` and targeting container port `8505`. It selects pods labelled `app: enterprise-rbac-chatbot`. There is no Ingress, Gateway, LoadBalancer, or NodePort manifest in the current repository. External browser access therefore requires an additional exposure mechanism or temporary port-forwarding.

### Qdrant service and storage

The `qdrant` Service exposes HTTP 6333 and gRPC 6334 inside the cluster. The Qdrant Deployment mounts `qdrant-pvc` at `/qdrant/storage`. The claim requests 2Gi with `ReadWriteOnce`. No StorageClass is specified; provisioning depends on the cluster's default StorageClass or a matching pre-provisioned PersistentVolume.

## Configuration and secrets

### ConfigMap

The `chatbot-config` ConfigMap currently defines:

| Key | Configured value | Purpose |
|---|---|---|
| `QDRANT_URL` | `http://qdrant:6333` | In-cluster Qdrant endpoint |
| `QDRANT_COLLECTION` | `company_docs` | Vector collection |
| `EMBEDDING_MODEL` | `sentence-transformers/all-MiniLM-L6-v2` | Embedding model |
| `GROQ_MODEL` | `openai/gpt-oss-20b` | Chat model identifier |
| `LANGCHAIN_TRACING_V2` | `true` | Enables LangChain tracing |
| `LANGCHAIN_ENDPOINT` | `https://api.smith.langchain.com` | LangSmith endpoint |
| `LANGCHAIN_PROJECT` | `enterprise-rbac-chatbot` | LangSmith project |

These non-sensitive values are injected through `envFrom.configMapRef`.

### Kubernetes Secret

The Rollout references a Secret named `chatbot-secrets` using `envFrom.secretRef`. The application source expects these keys:

- `GROQ_API_KEY` — credential for the Groq integration.
- `CHATBOT_USERS_JSON` — JSON user directory containing bcrypt password hashes, departments, and roles.

The Secret is not a file in this repository and no secret values should be committed to Git. Create it in `enterprise-chatbot` before expecting chatbot pods to start successfully.

For example, with the required value already available in your shell and a protected `users.json` file:

```bash
kubectl create namespace enterprise-chatbot --dry-run=client -o yaml | kubectl apply -f -

kubectl create secret generic chatbot-secrets \
  -n enterprise-chatbot \
  --from-literal=GROQ_API_KEY="$GROQ_API_KEY" \
  --from-file=CHATBOT_USERS_JSON=./users.json \
  --dry-run=client -o yaml | kubectl apply -f -
```

Keep `users.json` out of source control and restrict access to the Secret. This command creates or updates the Secret without printing its values.

## Prerequisites

1. A reachable Kubernetes cluster and working `kubectl` context.
2. Argo CD installed, with its API resources available and the `argocd` namespace present.
3. Argo Rollouts controller and CRDs installed. The chatbot is an `argoproj.io/v1alpha1` Rollout, not a standard Deployment.
4. Kustomize support (included in recent `kubectl` versions).
5. A cluster StorageClass capable of provisioning the Qdrant 2Gi PVC, or a matching PersistentVolume.
6. The `chatbot-secrets` Secret created in `enterprise-chatbot`.
7. A published image at `shaikhdocker/enterprise-rbac-chatbot` matching the tag in `app/kustomization.yaml`.

Check the environment:

```bash
kubectl config current-context
kubectl get nodes
kubectl get namespace argocd
kubectl get crd rollouts.argoproj.io
kubectl kustomize app
```

This repository does not pin Kubernetes, Argo CD, Argo Rollouts, or Kustomize versions. Confirm compatibility with versions installed in your cluster.

## Deploy with Argo CD

### 1. Review the rendered manifests

From the repository root:

```bash
kubectl kustomize app
```

Review the output. This renders the application resources; it does not create the Argo CD Application itself.

### 2. Create the Secret

Follow the [Kubernetes Secret](#kubernetes-secret) instructions. It must exist in the destination namespace `enterprise-chatbot`.

### 3. Register the Argo CD Application

Apply the Application manifest:

```bash
kubectl apply -f argocd/application.yaml
```

The current `argocd/application.yaml` specifies:

| Setting | Value |
|---|---|
| Application name | `enterprise-rbac-chatbot` |
| Argo CD namespace | `argocd` |
| Project | `default` |
| Source repository | `https://github.com/shaikh4git/Enterprise-RBAC-Chatbot-GitOps.git` |
| Target revision | `main` |
| Source path | `app` |
| Destination | `https://kubernetes.default.svc` |
| Destination namespace | `enterprise-chatbot` |
| Automated sync | Enabled |
| Prune | Enabled |
| Self-heal | Enabled |
| Sync option | `CreateNamespace=true` |

The Application creates the destination namespace if needed, and `app/namespace.yaml` also declares it.

### 4. Inspect and synchronize

```bash
argocd app get enterprise-rbac-chatbot
argocd app sync enterprise-rbac-chatbot
argocd app wait enterprise-rbac-chatbot --health
```

The Argo CD CLI must be authenticated to the correct instance. Since automated sync is enabled, manual sync is normally unnecessary once the Application is configured and healthy.

## Build and promote an application image

The application source and Dockerfile live in the [application repository](https://github.com/shaikh4git/Enterprise-RBAC-Chatbot). This GitOps repository does not build the image; it selects the published image tag Kubernetes should deploy.

Image name:

```text
shaikhdocker/enterprise-rbac-chatbot
```

The current `app/kustomization.yaml` sets:

```yaml
images:
  - name: shaikhdocker/enterprise-rbac-chatbot
    newTag: 6c0cfd3
```

To promote a new build:

1. Build and publish the application image from the application repository.
2. Choose a tag that was actually pushed to the registry.
3. Update `newTag` in `app/kustomization.yaml` to that exact tag.
4. Commit and push the change to `main` in this GitOps repository.
5. Review Argo CD sync/health and observe the Rollout.

The image transformer applies the tag to the matching image reference in the Rollout manifest. Argo CD then reconciles the rendered manifests.

Prefer immutable build tags (for example, a Git commit SHA) over reusing mutable tags. The Rollout specifies `imagePullPolicy: IfNotPresent`, so new immutable tags help avoid ambiguity from cached images.

## Canary rollout operations

The chatbot is managed by an Argo Rollouts resource named `enterprise-rbac-chatbot`, with two desired replicas and a revision history limit of three.

The configured strategy is:

```yaml
strategy:
  canary:
    maxSurge: 1
    maxUnavailable: 0
    steps:
      - setWeight: 50
      - pause: {}
      - setWeight: 100
```

At a high level:

1. Argo Rollouts begins the canary update and sets the canary step weight to 50.
2. The rollout pauses at the manual `pause: {}` step for operator review.
3. An operator inspects the rollout and either promotes or aborts it.
4. Promotion advances the rollout to the 100% step and completes the update.

The strategy allows up to one extra pod above the desired count (`maxSurge: 1`) and sets `maxUnavailable: 0`.

### Traffic-routing limitation

The current manifests define one chatbot ClusterIP Service, `enterprise-rbac-chatbot`, selecting the common label `app: enterprise-rbac-chatbot`. They do not define separate stable/canary Services or an explicit `trafficRouting` integration (for example, an ingress or service-mesh router).

Therefore, the `setWeight` values are Argo Rollouts canary progression settings, not a guarantee that an exact percentage of end-user requests reaches the new version. With the current single-Service setup, request distribution depends on ready endpoints and Kubernetes Service load balancing. Add an explicit traffic router and stable/canary service topology if precise request-level traffic shifting is required.

### Inspect, promote, or abort

Install and configure the `kubectl-argo-rollouts` plugin:

```bash
kubectl argo rollouts get rollout enterprise-rbac-chatbot -n enterprise-chatbot
kubectl argo rollouts status enterprise-rbac-chatbot -n enterprise-chatbot
kubectl argo rollouts promote enterprise-rbac-chatbot -n enterprise-chatbot
kubectl argo rollouts abort enterprise-rbac-chatbot -n enterprise-chatbot
```

Promote only after reviewing the canary. Abort an unsatisfactory update, then inspect the Rollout and Argo CD state.

## Verify the deployment

### Argo CD

```bash
argocd app get enterprise-rbac-chatbot
argocd app diff enterprise-rbac-chatbot
argocd app history enterprise-rbac-chatbot
```

### Kubernetes resources

```bash
kubectl get all -n enterprise-chatbot
kubectl get rollout enterprise-rbac-chatbot -n enterprise-chatbot
kubectl get pods -n enterprise-chatbot -o wide
kubectl get services -n enterprise-chatbot
kubectl get pvc -n enterprise-chatbot
kubectl get secret chatbot-secrets -n enterprise-chatbot
```

The last command reports Secret metadata, not its values. Do not print Secret YAML or decode Secret data in shared logs.

### Events and logs

```bash
kubectl get events -n enterprise-chatbot --sort-by=.lastTimestamp
kubectl describe rollout enterprise-rbac-chatbot -n enterprise-chatbot
kubectl describe pod -n enterprise-chatbot <pod-name>
kubectl logs -n enterprise-chatbot <pod-name> -c chatbot
kubectl logs -n enterprise-chatbot deployment/qdrant
```

Replace `<pod-name>` with an actual pod name from `kubectl get pods`.

### Temporarily access Streamlit

The chatbot Service is internal-only. To access it from your workstation:

```bash
kubectl port-forward -n enterprise-chatbot service/enterprise-rbac-chatbot 8505:8505
```

Then open http://localhost:8505. Stop the port-forward with Ctrl+C.

## Troubleshooting

| Symptom | Checks |
|---|---|
| Argo CD Application is missing or cannot sync | Confirm Argo CD is installed, `argocd` namespace exists, and `argocd/application.yaml` was applied. Check repository access, `main` branch, and `app` path. |
| Rollout kind is not recognized | Confirm Argo Rollouts CRDs and controller are installed and healthy. |
| Chatbot pod reports a missing Secret | Create `chatbot-secrets` in `enterprise-chatbot` with the required keys. |
| Chatbot pod is in ImagePullBackOff | Confirm the image exists, the tag was pushed, and the cluster can pull from the registry. |
| Chatbot pod is not Ready | Inspect pod events and logs; the readiness probe checks `/_stcore/health` on port 8505. |
| Qdrant pod is pending | Check PVC status, StorageClass availability, and cluster capacity. |
| Qdrant pod is not Ready | Inspect Qdrant logs and events; probes use TCP port 6333. |
| Chatbot cannot reach Qdrant | Confirm the `qdrant` Service and endpoints exist and `QDRANT_URL` is `http://qdrant:6333`. |
| Rollout remains paused | Inspect it with `kubectl argo rollouts get rollout enterprise-rbac-chatbot -n enterprise-chatbot`; promote only after review. |

## Operational notes and known limitations

These points reflect the manifests currently present in this repository:

- **No external ingress is configured.** The chatbot Service is ClusterIP-only.
- **The Secret is external to Git.** `chatbot-secrets` is referenced but is not generated by Kustomize.
- **No explicit traffic router is configured.** The canary steps alone do not guarantee request-level 50/50 traffic splitting.
- **Qdrant storage requests 2Gi.** The claim does not select a StorageClass; provisioning depends on the cluster.
- **Qdrant runs as one replica.** No Qdrant high-availability topology is defined.
- **No NetworkPolicy, Ingress, Gateway, HPA, PodDisruptionBudget, or monitoring resources** are present in the current Kustomize resource list.
- **Argo CD automated pruning is enabled.** Removing a resource from the Kustomize output can cause Argo CD to prune it. Review removals carefully.
- **The chatbot uses a non-root security context.** The Rollout runs as UID 10001, disallows privilege escalation, and drops Linux capabilities.
- **The model cache uses emptyDir.** The Hugging Face cache is ephemeral and may need to be populated again when pods are recreated.

Before production use, review availability, backup and restore, network exposure, secret management, resource sizing, observability, and canary traffic routing for your environment.

## Related project

- [Enterprise RBAC Chatbot — application source, Dockerfile, authentication, RBAC, ingestion and RAG](https://github.com/shaikh4git/Enterprise-RBAC-Chatbot)
- [Enterprise RBAC Chatbot GitOps — Kubernetes and Argo CD deployment](https://github.com/shaikh4git/Enterprise-RBAC-Chatbot-GitOps)
