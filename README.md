# AnythingLLM on OpenShift

AnythingLLM is an open-source AI document chat and RAG (Retrieval-Augmented Generation) platform that can utilize RHOAI resources for LLM and embedding capabilities.

## Overview

This deployment includes:
- **AnythingLLM** - Document chat interface with RAG support
- **LanceDB** - Built-in vector database for embeddings
- **OAuth Proxy** - OpenShift authentication integration
- **RHOAI Integration** - Uses RHOAI InferenceServices for LLM and embeddings

## Prerequisites

- OpenShift 4.19+ with admin access
- RHOAI deployed with at least one InferenceService
- Helm 3 installed (for direct deployment)
- OpenShift GitOps/ArgoCD (for GitOps deployment)

### Install OpenShift GitOps (if needed)

```bash
./bootstrap.sh
```

**Note:** If GitOps is already installed (e.g., from deploying another repository), the bootstrap script will detect it and skip installation.

## Deployment Options

### Option 1: GitOps Deployment with ArgoCD (Recommended)

**Step 1: Find your RHOAI InferenceService URLs**

```bash
# List all InferenceServices
oc get inferenceservices -A

# Get internal cluster URL (format: http://<model-name>-predictor.<namespace>.svc.cluster.local/v1)
oc get inferenceservice granite-7b -n demo-models -o jsonpath='{.status.components.predictor.url}'
```

**Step 2: Update the ArgoCD Application**

Edit `gitops/anythingllm.yaml` to update the repoURL if needed, then apply:

```bash
oc apply -f gitops/anythingllm.yaml
```

**Step 3: Configure model endpoints**

After deployment, you need to configure AnythingLLM with your RHOAI endpoints through the web UI:

1. Get the route URL:
   ```bash
   oc get route anythingllm -n demo-apps -o jsonpath='{.spec.host}'
   ```

2. Access the UI and complete the setup wizard
3. Configure LLM settings:
   - Provider: `Generic OpenAI`
   - Base URL: `http://granite-7b-predictor.demo-models.svc.cluster.local/v1`
   - Model: `granite-7b-instruct`
   - API Key: (leave empty for internal cluster access)

4. Configure embedding settings:
   - Provider: `Generic OpenAI`
   - Base URL: `http://granite-7b-predictor.demo-models.svc.cluster.local/v1`
   - Model: `granite-7b-instruct`

**Step 4: Monitor deployment**

```bash
# Watch ArgoCD application status
oc get application anythingllm -n openshift-gitops -w

# Check pod status
oc get pods -n demo-apps
```

### Option 2: Direct Helm Deployment

**Step 1: Configure model endpoints**

Edit `helm/values-rhoai.yaml` and update the environment variables with your RHOAI InferenceService URLs:

```yaml
env:
  - name: GENERIC_OPEN_AI_BASE_PATH
    value: "http://granite-7b-predictor.demo-models.svc.cluster.local/v1"
  - name: GENERIC_OPEN_AI_MODEL_PREF
    value: "granite-7b-instruct"
  - name: EMBEDDING_BASE_PATH
    value: "http://granite-7b-predictor.demo-models.svc.cluster.local/v1"
  - name: EMBEDDING_MODEL_PREF
    value: "granite-7b-instruct"
```

**Step 2: Deploy with Helm**

```bash
helm install anythingllm ./helm \
  -f ./helm/values-rhoai.yaml \
  -n demo-apps --create-namespace
```

**Step 3: Access AnythingLLM**

```bash
# Get the route URL
oc get route anythingllm -n demo-apps -o jsonpath='{.spec.host}'

# Open in browser (you'll be prompted to log in with OpenShift OAuth)
open https://$(oc get route anythingllm -n demo-apps -o jsonpath='{.spec.host}')
```

## Verification

```bash
# Check pod status
oc get pods -n demo-apps

# View logs
oc logs -n demo-apps deployment/anythingllm -c anythingllm --tail=50

# Test RHOAI connection from pod
oc exec -n demo-apps deployment/anythingllm -c anythingllm -- \
  curl -s http://granite-7b-predictor.demo-models.svc.cluster.local/v1/models
```

## Configuration

### Storage

Default: 10Gi PVC for document storage and vector database

To change storage size, edit `helm/values-rhoai.yaml`:

```yaml
storage:
  size: 20Gi
```

### Resources

Adjust CPU and memory in `helm/values-rhoai.yaml`:

```yaml
resources:
  requests:
    cpu: "1"
    memory: "2Gi"
  limits:
    cpu: "4"
    memory: "8Gi"
```

### Vector Database

Default: LanceDB (built-in)

To use a different vector database, update `helm/values-rhoai.yaml`:

```yaml
vectorDatabase:
  type: "chroma"  # or pinecone, weaviate, etc.
```

## Features

- Document upload and chat (PDF, DOCX, TXT, etc.)
- RAG with vector search
- Workspace management
- Multi-user support with OpenShift OAuth
- Integration with RHOAI InferenceServices
- Built-in vector database (LanceDB)

## Troubleshooting

### Pod not starting
```bash
oc describe pod -n demo-apps -l app=anythingllm
oc logs -n demo-apps deployment/anythingllm -c anythingllm --tail=100
```

### Can't connect to RHOAI
```bash
# Verify cross-namespace RBAC
oc get rolebinding anythingllm-inferenceservice-access -n demo-models

# Test connection from pod
oc exec -n demo-apps deployment/anythingllm -c anythingllm -- \
  curl -v http://granite-7b-predictor.demo-models.svc.cluster.local/v1/models
```

### Storage issues
```bash
# Check PVC status
oc get pvc -n demo-apps

# Describe PVC for events
oc describe pvc anythingllm-storage -n demo-apps
```

## Uninstall

```bash
# With Helm
helm uninstall anythingllm -n demo-apps

# With GitOps
oc delete -f gitops/anythingllm.yaml

# Delete PVC (optional)
oc delete pvc anythingllm-storage -n demo-apps
```

## Support

- AnythingLLM: https://github.com/Mintplex-Labs/anything-llm
- RHOAI: https://access.redhat.com/products/red-hat-openshift-ai
