# SnapCD Runner - Kubernetes Deployment

This guide explains how to deploy the SnapCD Runner on Kubernetes using Kustomize.

Please note that the below a manual, locally executed deployment and not suitable for automation. For a deployment that used the `kustomize` terraform provider configure and deploy the manifests found in this repo, see: 


## Prerequisites

- Kubernetes cluster (1.19+)
- kubectl configured to access your cluster
- Kustomize (built into kubectl 1.14+)
- Network access from the cluster to [snapcd.io](https://snapcd.io)
- Organization ID and Runner ID, as well as Service Principal credentials (Client ID and Client Secret) from [snapcd.io](https://snapcd.io)

## Architecture

This deployment creates:

- **StatefulSet** - Ensures stable network identity and persistent storage for each runner replica
- **ConfigMaps** - For appsettings.json, known_hosts, and preapproved hooks
- **Secrets** - For SSH private key and credentials (ClientSecret)
- **PersistentVolumeClaim** - For runner working directory (one per replica)

The init container automatically downloads Terraform and OpenTofu binaries to an ephemeral volume shared with the main container.

Each StatefulSet replica uses its pod name (e.g., `snapcd-runner-0`) as the Runner Instance name.

## Directory Structure

```
manifests/
├── base/                    # Base manifests (templates, do not edit for secrets)
│   ├── kustomization.yaml
│   ├── namespace.yaml
│   ├── statefulset.yaml
│   └── config/              # Example/template config files
│       ├── appsettings.json
│       ├── known_hosts
│       ├── id_rsa
│       ├── secret.env.example
│       └── preapproved-hooks/
└── production/              # Your actual deployment (config files gitignored)
    ├── kustomization.yaml
    └── config/
        ├── appsettings.json   # Your actual config
        ├── known_hosts        # Your known_hosts
        ├── id_rsa             # Your SSH key
        └── secret.env         # Your secrets
```

## Configuration

All configuration is done in the production directory. The base directory contains templates.

### 1. Initialize Config Files

The config files in `manifests/production/config/` are gitignored. Initialize them by copying from the base templates:

```bash
#!/bin/bash
# Run from the repository root

# Create the config directory
mkdir -p manifests/production/config/preapproved-hooks

# Copy template files from base
cp manifests/base/config/appsettings.json manifests/production/config/
cp manifests/base/config/known_hosts manifests/production/config/
cp manifests/base/config/id_rsa manifests/production/config/
cp manifests/base/config/secret.env.example manifests/production/config/secret.env
touch manifests/production/config/preapproved-hooks/.gitkeep

echo "Config files initialized. Edit them with your values:"
echo "  - manifests/production/config/appsettings.json"
echo "  - manifests/production/config/secret.env"
echo "  - manifests/production/config/id_rsa (optional)"
echo "  - manifests/production/config/known_hosts (optional)"
```

Then edit the files with your actual values.

### 2. Application Settings

Edit `manifests/production/config/appsettings.json`:

```json
{
  "Runner": {
    "Id": "your-runner-id",
    "Instance": "0",
    "Credentials": {
      "ClientId": "your-client-id"
    },
    "OrganizationId": "your-organization-id"
  },
  "Server": {
    "Url": "https://snapcd.io"
  }
}
```

> Note: The `Instance` field is automatically overridden at runtime via the `Runner__Instance` environment variable, which is set to the pod name.

| Setting | Description |
|---------|-------------|
| `Runner.Id` | The unique ID of the runner (GUID format, obtained from SnapCD server) |
| `Runner.Instance` | Automatically set from pod name via environment variable |
| `Runner.Credentials.ClientId` | Service Principal client ID for authentication |
| `Runner.OrganizationId` | The organization ID this runner belongs to |
| `Server.Url` | The URL of your SnapCD server |

### 3. Credentials Secret

Edit `manifests/production/config/secret.env` with your actual secret:

```
Runner__Credentials__ClientSecret=your-actual-client-secret
```

> Note: All files in `manifests/production/config/` are gitignored. Never commit credentials to version control.

### 4. SSH Keys (Optional)

If your Terraform/OpenTofu modules use private Git repositories over SSH, replace the placeholder:

```bash
cp ~/.ssh/id_rsa manifests/production/config/id_rsa
```

The `known_hosts` file is pre-configured with GitHub and GitLab host keys. To add additional hosts:

```bash
ssh-keyscan your-git-server.com >> manifests/production/config/known_hosts
```

### 5. Engine Versions

The init container downloads Terraform and OpenTofu. To change versions, edit `manifests/base/statefulset.yaml`:

```yaml
env:
  - name: TOFU_VERSION
    value: "1.8.8"
  - name: TERRAFORM_VERSION
    value: "1.5.7"
```

> Please note that Snap CD strives to support the latest available version of `tofu`. For `terraform` we design for binaries up to release [1.5.7](https://github.com/hashicorp/terraform/releases/tag/v1.5.7), which was the final release under the [Mozilla Public License 2.0](https://github.com/hashicorp/terraform/blob/v1.5.7/LICENSE). It was not developed for later versions, such as those published under the [Business Source License 1.1 (BSL 1.1)](https://github.com/hashicorp/terraform/blob/v1.6.0/LICENSE).

### 6. Pre-approved Hooks (Optional)

Place approved hook scripts in `manifests/base/config/preapproved-hooks/` and enable in appsettings.json:

```json
{
  "HooksPreapproval": {
    "Enabled": true,
    "PreapprovedHooksDirectory": "/app/preapproved-hooks"
  }
}
```

### 7. Scaling

To run multiple runner replicas, create a patch in your overlay or edit `manifests/base/statefulset.yaml`:

```yaml
spec:
  replicas: 3
```

Each replica will have its own persistent volume and unique instance name (snapcd-runner-0, snapcd-runner-1, snapcd-runner-2).

## Deployment

### Deploy with Kustomize

```bash
# Preview the manifests
kubectl kustomize manifests/production

# Apply to cluster
kubectl apply -k manifests/production
```

### Verify Deployment

```bash
# Check pod status
kubectl get pods -n snapcd

# View logs
kubectl logs -n snapcd snapcd-runner-0

# Check all resources
kubectl get all -n snapcd
```

### Delete Deployment

```bash
kubectl delete -k manifests/production
```

> Note: PersistentVolumeClaims are not deleted automatically. To remove them:
> ```bash
> kubectl delete pvc -n snapcd -l app=snapcd-runner
> ```

## Multiple Environments

Create additional environment directories:

```
manifests/
├── base/
├── production/
│   ├── kustomization.yaml
│   └── config/...
├── staging/
│   ├── kustomization.yaml
│   └── config/...
└── dev/
    ├── kustomization.yaml
    └── config/...
```

Deploy to a specific environment:

```bash
kubectl apply -k manifests/staging
```

## Docker Image

This deployment uses the `ghcr.io/schrieksoft/snapcd-runner:azure-0.1.4` image, which includes:

- .NET 10 runtime
- Git and SSH client
- Azure CLI (in order to use the `azurerm` and `azuread` terraform providers)

## Troubleshooting

### Check init container logs

```bash
kubectl logs -n snapcd snapcd-runner-0 -c download-engines
```

### Exec into running container

```bash
kubectl exec -it -n snapcd snapcd-runner-0 -- /bin/bash
```

### Azure Authentication

If using Azure resources, exec into the pod and authenticate:

```bash
kubectl exec -it -n snapcd snapcd-runner-0 -- /bin/bash
az login
```