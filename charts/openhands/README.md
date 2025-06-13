# Openhands-all Helm Chart

This Helm chart deploys the complete OpenHands stack, including all required dependencies. It's designed to be a one-stop solution for deploying OpenHands in a Kubernetes environment.

## Prerequisites

- Kubernetes 1.19+
- Helm 3.2.0+
- PV provisioner support in the underlying infrastructure (if persistence is enabled)
- Ingress controller (recommended: Traefik)
- A TLS solution for certificates. We use cert-manager to handle this in cluster normally.
- Storage class named "standard-rwo" (or configure a different storage class name as described below)

### Storage Class Configuration

By default, the chart expects a storage class named `standard-rwo`. If you're using EKS, which typically has a `gp2` storage class, you can configure the chart to use it instead:

```yaml
runtime-api:
  env:
    STORAGE_CLASS: "gp2"  # Replace with your cluster's storage class name
```

Alternatively, you can create a storage class named `standard-rwo` that uses your cloud provider's block storage:

```bash
# For AWS EKS
kubectl apply -f - <<EOF
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: standard-rwo
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp2
volumeBindingMode: WaitForFirstConsumer
EOF

# For GKE
kubectl apply -f - <<EOF
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: standard-rwo
provisioner: pd.csi.storage.gke.io
parameters:
  type: pd-standard
volumeBindingMode: WaitForFirstConsumer
EOF
```

## Installation
### Initial setup
1. Clone the repository and navigate to the chart directory:
   ```bash
   git clone https://github.com/All-Hands-AI/on-prem.git
   cd on-prem/k8s/helm/openhands-all
   ```

1. Create required, non-keycloak secrets:
   ```bash
   # Create namespace (if you want to use a different namespace, you will need to change it in all the commands that follow)
   kubectl create namespace openhands

   # Create JWT secret for sessions
   kubectl create secret generic jwt-secret -n openhands --from-literal=jwt-secret=<your-jwt-secret>

   # Create PostgreSQL password secret
   kubectl create secret generic postgres-password -n openhands \
     --from-literal=username=postgres \
     --from-literal=password=<your-postgres-password> \
     --from-literal=postgres-password=<your-postgres-password>

   # Create Redis password secret
   kubectl create secret generic redis -n openhands \
     --from-literal=redis-password=<your-redis-password>

   # Create LiteLLM API key secret
   kubectl create secret generic lite-llm-api-key -n openhands \
     --from-literal=lite-llm-api-key=<your-litellm-api-key>

   # Create langfuse salt secret
   kubectl create secret generic langfuse-salt -n openhands \
     --from-literal=salt=<your-salt-value>

   # Create langfuse nextauth secret (for signing JWTs)
   kubectl create secret generic langfuse-nextauth -n openhands \
     --from-literal=nextauth-secret=<your-nextauth-secret>

   # Create clickhouse password secret
   kubectl create secret generic clickhouse-password -n openhands \
     --from-literal=password=<your-clickhouse-password>

   # Create two secrets, one that configures runtime-api with a default api key, and another that configures openhands to utilize that api key.
   # There are two secrets because these two workloads can also run in different clusters/namespaces
   kubectl create secret generic default-api-key -n openhands \
     --from-literal=default-api-key=<your-runtime-api-key>

   kubectl create secret generic sandbox-api-key -n openhands \
     --from-literal=sandbox-api-key=<your-runtime-api-key>
   ```

  If your litellm configuration will use environment variables to hold your LLM API Keys, you will configure them as key/value pairs in the secret `litellm-env-secrets`. Even if you don't need these environment variables, you should set the following in your values:
  ```yaml
  litellm-helm:
    environmentSecrets: []
  ```

  Example secret:
  ```bash
  kubectl create secret generic litellm-env-secrets -n openhands \
     --from-literal=ANTHROPIC_API_KEY=<your-anthropic-api-key>
  ```

### Keycloak with GitHub OAuth Configuration

The chart includes Keycloak for authentication and supports GitHub OAuth integration. To configure GitHub OAuth:

1. Create a GitHub OAuth App:
   - Go to your GitHub organization settings or personal settings
   - Navigate to "Developer settings" > "OAuth Apps" > "New OAuth App"
   - Set the "Authorization callback URL" to `https://<your-keycloak-hostname>/realms/<realm-name>/broker/github/endpoint`
   - Note the Client ID and Client Secret provided by GitHub

1. Create a Kubernetes secret for Keycloak realm configuration:
   ```bash
   kubectl create secret generic keycloak-realm -n openhands \
     --from-literal=client-id=allhands \
     --from-literal=client-secret=<your-keycloak-client-secret> \
     --from-literal=provider-name=github \
     --from-literal=realm-name=allhands \
     --from-literal=server-url=http://keycloak
   ```

   The secret must contain the following keys:
   - `client-id`: The Keycloak client ID (not the GitHub OAuth App client ID), set to "allhands"
   - `client-secret`: The Keycloak client secret (not the GitHub OAuth App client secret)
   - `provider-name`: Set to "github" for GitHub OAuth integration
   - `realm-name`: The name of your Keycloak realm, set to "allhands"
   - `server-url`: The internal Keycloak service URL (typically "http://keycloak")

1. Create a GitHub App secret with the following structure:
   This secret contains the GitHub App configuration information from your GitHub account.
   You can create it using kubectl:
   ```bash
   kubectl create secret generic github-app -n openhands \
     --from-literal=app-id=<your-github-app-id> \
     --from-literal=webhook-secret=<your-github-webhook-secret> \
     --from-literal=client-id=<your-github-client-id> \
     --from-literal=client-secret=<your-github-client-secret> \
     --from-file=private-key=<path-to-your-private-key-file>
   ```

1. Create a Keycloak admin password secret:
   ```bash
   kubectl create secret generic keycloak-admin -n openhands \
     --from-literal=admin-password=<your-keycloak-admin-password>
   ```

   This secret contains the password for the Keycloak admin user. It's used by the configuration job to set up the Keycloak realm and identity providers.

1. Configure Keycloak in your values.yaml:
   ```yaml
   keycloak:
     enabled: true
     url: "https://<your-keycloak-hostname>"
     ingress:
       enabled: true
       hostname: <your-keycloak-hostname>
   ```

When the chart is deployed, a job will run to configure the Keycloak realm with GitHub as an identity provider using the credentials you provided.

### Install OpenHands
Install the chart!

```bash
helm upgrade --install openhands --namespace openhands . -f my-values.yaml
```

### Configure lite-llm team ID in openhands values
After installing the chart initially you will want to port-forward to litellm and create a new team.

```bash
kubectl port-forward svc/openhands-litellm 4000:4000
```

Navigate to http://localhost:4000/ui in your browser and login using the `admin` username and the API key from `lite-llm-api-key` secret you configured earlier.
Once logged in, go to Teams -> Click Create New Team. Name it whatever you want and then get the numerical team id. In your values file, update the following and then upgrade the helm install with the new value.

values-update:
```yaml
litellm:
  ...
  teamId: "<UPDATE_WITH_TEAM_ID>"
  ...
```

Upgrade the release:
```bash
helm upgrade --install openhands --namespace openhands . -f my-values.yaml
```

## Configuration

See the [values.yaml](values.yaml) file for the full list of configurable parameters. Make sure to update all values marked with "REQUIRED" comments.

An [example-values.yaml](example-values.yaml) file is also provided as a starting point for your own configuration. This example file contains the minimum set of values you need to override when deploying the chart with the default included services (without using external data stores). Remember to update the domain names and other environment-specific values in the example file before using it.

## Using External Services

### Bring Your Own PostgreSQL

To use an external PostgreSQL database instead of deploying one with the chart:

1. Disable the included PostgreSQL:
   ```yaml
   postgresql:
     enabled: false
   ```

2. Configure the external database connection:
   ```yaml
   externalDatabase:
     host: your-postgresql-host
     port: 5432
     database: openhands
     existingSecret: postgres-password

   # Make sure the secret exists with the correct credentials
   # kubectl create secret generic postgres-password \
   #   --from-literal=username=<your-db-username> \
   #   --from-literal=password=<your-db-password>
   ```

3. Update the Keycloak, LiteLLM, and runtime-api configurations to use the external database:
   ```yaml
   keycloak:
     externalDatabase:
       host: your-postgresql-host
       port: 5432
       existingSecret: postgres-password

   litellm-helm:
     db:
       deployStandalone: false
       useExisting: true
       database: litellm
       endpoint: your-postgresql-host
       secret:
         name: postgres-password

   runtime-api:
     postgresql:
       auth:
         existingSecret: postgres-password
     env:
       DB_HOST: your-postgresql-host
       DB_USER: your-db-username
       DB_NAME: runtime_api_db
   ```

### Bring Your Own Redis

To use an external Redis instance:

1. Disable the included Redis:
   ```yaml
   redis:
     enabled: false
   ```

2. Configure the external Redis connection:
   ```yaml
   externalRedis:
     host: your-redis-host
     port: 6379
     existingSecret: redis

   # Make sure the secret exists with the correct credentials
   # kubectl create secret generic redis \
   #   --from-literal=redis-password=<your-redis-password>
   ```

### Bring Your Own S3-Compatible Storage

To use an external S3-compatible storage instead of MinIO:

1. Disable the ephemeral filestore:
   ```yaml
   filestore:
     ephemeral: false
   ```

2. Configure the S3 connection:
   ```yaml
   filestore:
     ephemeral: false
     bucket: your-bucket-name
     endpoint: https://your-s3-endpoint
     region: your-s3-region
     existingSecret: s3-credentials

   # Make sure the secret exists with the correct credentials
   # kubectl create secret generic s3-credentials \
   #   --from-literal=access-key=<your-access-key> \
   #   --from-literal=secret-key=<your-secret-key>
   ```

## Upgrading

To upgrade the chart:

```bash
helm upgrade openhands . -f my-values.yaml -n openhands
```

## Uninstallation

To uninstall the chart:

```bash
helm uninstall openhands -n openhands
```

Note: This will not delete any PVCs or secrets created. You'll need to delete those manually if desired.
