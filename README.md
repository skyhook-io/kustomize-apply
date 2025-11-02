# Kustomize Apply

Apply kustomize overlays to Kubernetes clusters with robust metadata extraction and precise workload tracking.

## Features

- 🚀 **Direct apply** - kubectl apply with kustomize
- ⏳ **Precise wait** - Only waits for workloads in your manifests
- 📦 **Namespace management** - Auto-create namespaces
- 🔍 **Dry run** - Preview changes before applying
- 🎯 **JSON outputs** - Structured data for downstream use
- 🔐 **Self-contained** - Optional kubeconfig support

## Usage

```yaml
# Typical usage with kustomize-inspect
- name: Inspect overlay
  uses: skyhook-io/kustomize-inspect@v1
  id: inspect
  with:
    overlay_dir: deploy/overlays/production

- name: Apply to cluster
  uses: skyhook-io/kustomize-apply@v1
  with:
    overlay_dir: deploy/overlays/production
    namespace: ${{ steps.inspect.outputs.namespace }}
    workloads_json: ${{ steps.inspect.outputs.workloads_json }}
```

## Inputs

| Input | Description | Required | Default |
|-------|-------------|----------|---------|
| `overlay_dir` | Path to kustomize overlay | ✅ | - |
| `namespace` | Target namespace (from Inspect) | ✅ | - |
| `workloads_json` | Workloads to track (from Inspect) | ❌ | `[]` |
| `dry_run` | Server-side dry run only | ❌ | `false` |
| `validate` | Client-side schema validation | ❌ | `true` |
| `server_side` | Use Server-Side Apply | ❌ | `false` |
| `wait` | Wait for workloads to be ready | ❌ | `true` |
| `wait_timeout` | Wait timeout in seconds | ❌ | `300` |

## Outputs

| Output | Description | Example |
|--------|-------------|---------|
| `applied_resources_json` | kubectl apply output as JSON | `["deployment.apps/api configured"]` |

## Examples

### Basic deployment with inspect
```yaml
- name: Edit kustomization
  uses: skyhook-io/kustomize-edit@v1
  with:
    overlay_dir: deploy/overlays/staging
    image: backend
    tag: v1.2.3

- name: Inspect changes
  uses: skyhook-io/kustomize-inspect@v1
  id: inspect
  with:
    overlay_dir: deploy/overlays/staging

- name: Apply to cluster
  uses: skyhook-io/kustomize-apply@v1
  with:
    overlay_dir: deploy/overlays/staging
    namespace: ${{ steps.inspect.outputs.namespace }}
    workloads_json: ${{ steps.inspect.outputs.workloads_json }}
```

### Dry run first
```yaml
- name: Inspect
  uses: skyhook-io/kustomize-inspect@v1
  id: inspect
  with:
    overlay_dir: deploy/overlays/production

- name: Preview changes
  uses: skyhook-io/kustomize-apply@v1
  with:
    overlay_dir: deploy/overlays/production
    namespace: ${{ steps.inspect.outputs.namespace }}
    dry_run: true

- name: Apply if approved
  if: ${{ inputs.approved == 'true' }}
  uses: skyhook-io/kustomize-apply@v1
  with:
    overlay_dir: deploy/overlays/production
    namespace: ${{ steps.inspect.outputs.namespace }}
    workloads_json: ${{ steps.inspect.outputs.workloads_json }}
```

### No wait (fire and forget)
```yaml
- name: Quick apply
  uses: skyhook-io/kustomize-apply@v1
  with:
    overlay_dir: deploy/overlays/dev
    namespace: development
    wait: false
```

### Server-side apply
```yaml
- name: Apply with SSA
  uses: skyhook-io/kustomize-apply@v1
  with:
    overlay_dir: deploy/overlays/production
    namespace: ${{ steps.inspect.outputs.namespace }}
    server_side: true
```

## Complete Workflow Example

```yaml
jobs:
  deploy:
    steps:
      - name: Checkout
        uses: actions/checkout@v4
      
      - name: Login to cloud
        uses: skyhook-io/cloud-login@v1
        with:
          provider: aws
          region: us-east-1
          cluster: production
      
      - name: Update kustomization
        uses: skyhook-io/kustomize-edit@v1
        with:
          overlay_dir: deploy/overlays/production
          image: backend
          tag: ${{ inputs.tag }}
          annotations: |
            deployed-by:${{ github.actor }}
            deployment-id:${{ github.run_id }}
      
      - name: Inspect changes
        uses: skyhook-io/kustomize-inspect@v1
        id: inspect
        with:
          overlay_dir: deploy/overlays/production
      
      - name: Apply to cluster
        uses: skyhook-io/kustomize-apply@v1
        with:
          overlay_dir: deploy/overlays/production
          namespace: ${{ steps.inspect.outputs.namespace }}
          workloads_json: ${{ steps.inspect.outputs.workloads_json }}
          wait: true
          wait_timeout: 300
      
      - name: Show deployment info
        run: |
          echo "Deployed to namespace: ${{ steps.inspect.outputs.namespace }}"
          echo "Primary deployment: ${{ steps.inspect.outputs.primary_deployment }}"
```

## How It Works

1. **Builds** manifests using `kustomize build`
2. **Validates** against kubectl schema (optional)
3. **Ensures** namespace exists
4. **Applies** manifests to cluster
5. **Waits** only for workloads provided via `workloads_json`

The action uses robust YAML parsing with `yq` to precisely track only the workloads in your manifests, avoiding brittle grep-based extraction.

## Prerequisites

- `kubectl` configured with cluster access
- `kustomize` available
- `jq` for JSON processing
- Appropriate RBAC permissions

## Notes

- Designed to work with kustomize-inspect for metadata
- No force or prune options (by design)
- Waits only for workloads you're actually deploying
- Works with any valid kustomization
- Supports server-side apply for large manifests