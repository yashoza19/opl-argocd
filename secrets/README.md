# Secrets Management

This directory provides guidance and templates for managing sensitive credentials required for cluster provisioning with Hive and ACM.

## Required Secrets

The following secrets must be created in each cluster namespace before provisioning:

### 1. AWS Credentials (`aws-credentials`)

Required by Hive to provision infrastructure in AWS.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: aws-credentials
  namespace: <cluster-namespace>
type: Opaque
stringData:
  aws_access_key_id: YOUR_AWS_ACCESS_KEY_ID
  aws_secret_access_key: YOUR_AWS_SECRET_ACCESS_KEY
```

**Best Practice**: Create an AWS IAM user specifically for Hive with minimal required permissions. See [Hive AWS Permissions](https://github.com/openshift/hive/blob/master/docs/using-hive.md#aws) for the required IAM policy.

### 2. Pull Secret (`pull-secret`)

Required to pull OpenShift container images from Red Hat's registries.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: pull-secret
  namespace: <cluster-namespace>
type: kubernetes.io/dockerconfigjson
stringData:
  .dockerconfigjson: |
    {
      "auths": {
        "cloud.openshift.com": {
          "auth": "YOUR_ENCODED_AUTH",
          "email": "your-email@example.com"
        },
        "quay.io": {
          "auth": "YOUR_ENCODED_AUTH",
          "email": "your-email@example.com"
        },
        "registry.connect.redhat.com": {
          "auth": "YOUR_ENCODED_AUTH",
          "email": "your-email@example.com"
        },
        "registry.redhat.io": {
          "auth": "YOUR_ENCODED_AUTH",
          "email": "your-email@example.com"
        }
      }
    }
```

**How to obtain**: Download your pull secret from the [Red Hat Hybrid Cloud Console](https://console.redhat.com/openshift/install/pull-secret).

### 3. SSH Private Key (`<cluster-name>-ssh-key`)

Required for emergency access to cluster nodes.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: <cluster-name>-ssh-key
  namespace: <cluster-namespace>
type: Opaque
stringData:
  ssh-privatekey: |
    -----BEGIN OPENSSH PRIVATE KEY-----
    YOUR_PRIVATE_KEY_CONTENT_HERE
    -----END OPENSSH PRIVATE KEY-----
  ssh-publickey: ssh-rsa AAAA...YOUR_PUBLIC_KEY_HERE
```

**Generate a new key pair**:
```bash
ssh-keygen -t rsa -b 4096 -f cluster-ssh-key -N ""
```

## Secrets Management Strategies

### Option 1: Sealed Secrets (Recommended for GitOps)

[Sealed Secrets](https://github.com/bitnami-labs/sealed-secrets) allows you to encrypt secrets and store them safely in Git.

1. Install Sealed Secrets controller:
   ```bash
   oc apply -f https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.24.0/controller.yaml
   ```

2. Install kubeseal CLI:
   ```bash
   # macOS
   brew install kubeseal

   # Linux
   wget https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.24.0/kubeseal-linux-amd64
   sudo install -m 755 kubeseal-linux-amd64 /usr/local/bin/kubeseal
   ```

3. Create and seal a secret:
   ```bash
   # Create regular secret manifest
   kubectl create secret generic aws-credentials \
     --from-literal=aws_access_key_id=YOUR_KEY \
     --from-literal=aws_secret_access_key=YOUR_SECRET \
     --dry-run=client -o yaml > aws-credentials.yaml

   # Seal it
   kubeseal -f aws-credentials.yaml -w aws-credentials-sealed.yaml \
     --controller-namespace sealed-secrets \
     --controller-name sealed-secrets

   # The sealed secret can be safely committed to Git
   git add aws-credentials-sealed.yaml
   ```

4. Add sealed secrets to your cluster overlay:
   ```yaml
   # clusters/dev-cluster-01/kustomization.yaml
   resources:
     - ../../cluster-templates/aws-ha/base
     - cluster-config.yaml
     - aws-credentials-sealed.yaml
     - pull-secret-sealed.yaml
     - ssh-key-sealed.yaml
   ```

### Option 2: External Secrets Operator

[External Secrets Operator](https://external-secrets.io/) syncs secrets from external secret management systems.

Supports:
- AWS Secrets Manager
- Azure Key Vault
- HashiCorp Vault
- GCP Secret Manager

Example with AWS Secrets Manager:
```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: aws-credentials
  namespace: dev-cluster-01
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secret-store
    kind: SecretStore
  target:
    name: aws-credentials
    creationPolicy: Owner
  data:
    - secretKey: aws_access_key_id
      remoteRef:
        key: hive/aws-credentials
        property: access_key_id
    - secretKey: aws_secret_access_key
      remoteRef:
        key: hive/aws-credentials
        property: secret_access_key
```

### Option 3: Manual Secret Creation

For testing or small-scale deployments, create secrets manually:

```bash
# Create namespace
oc create namespace dev-cluster-01

# Create AWS credentials
oc create secret generic aws-credentials \
  --from-literal=aws_access_key_id=YOUR_KEY \
  --from-literal=aws_secret_access_key=YOUR_SECRET \
  -n dev-cluster-01

# Create pull secret
oc create secret generic pull-secret \
  --from-file=.dockerconfigjson=/path/to/pull-secret.json \
  --type=kubernetes.io/dockerconfigjson \
  -n dev-cluster-01

# Create SSH key secret
oc create secret generic dev-cluster-01-ssh-key \
  --from-file=ssh-privatekey=/path/to/ssh-key \
  --from-file=ssh-publickey=/path/to/ssh-key.pub \
  -n dev-cluster-01
```

## Security Best Practices

1. **Never commit unencrypted secrets to Git**
2. **Use separate AWS credentials per environment**
3. **Rotate credentials regularly**
4. **Use IAM roles with minimal permissions**
5. **Enable secret encryption at rest in etcd**
6. **Audit secret access using RBAC**
7. **Consider using a centralized secret management system for production**

## Secret Rotation

When rotating secrets:

1. Create new secret with updated credentials
2. Update ClusterDeployment to reference new secret (if name changed)
3. Delete old secret after verification
4. For AWS credentials, ensure old credentials remain valid until rotation completes

## Troubleshooting

### ClusterDeployment stuck in "Provisioning"

Check if secrets exist:
```bash
oc get secrets -n <cluster-namespace>
```

Verify secret content (base64 decode to verify):
```bash
oc get secret aws-credentials -n <cluster-namespace> -o jsonpath='{.data.aws_access_key_id}' | base64 -d
```

Check Hive controller logs:
```bash
oc logs -n hive deployment/hive-controllers -c hive-controllers --tail=100
```

### Invalid pull secret error

Validate pull secret JSON:
```bash
oc get secret pull-secret -n <cluster-namespace> -o jsonpath='{.data.\.dockerconfigjson}' | base64 -d | jq .
```

### SSH key not working

Ensure the key format is correct (OpenSSH format):
```bash
oc get secret <cluster-name>-ssh-key -n <cluster-namespace> -o jsonpath='{.data.ssh-privatekey}' | base64 -d | head -1
# Should output: -----BEGIN OPENSSH PRIVATE KEY-----
```

## Reference

- [Hive Documentation](https://github.com/openshift/hive/tree/master/docs)
- [Sealed Secrets](https://github.com/bitnami-labs/sealed-secrets)
- [External Secrets Operator](https://external-secrets.io/)
- [OpenShift Secrets Management](https://docs.openshift.com/container-platform/latest/security/encrypting-etcd.html)
