# Issuer Configuration Guides

After the cert-manager operator is deployed to managed clusters via the ACM add-on, you need to create Issuers or ClusterIssuers to actually issue certificates. This guide covers three common patterns.

The cert-manager operator runs in the `open-cluster-management-agent-addon` namespace (installed via OLM Subscription). ClusterIssuers are cluster-scoped and available to all namespaces.

---

## 1. Let's Encrypt (ACME)

### When to Use

- Public-facing services that need trusted TLS certificates
- Clusters with external DNS or ingress access
- Automated certificate renewal without manual intervention

### Prerequisites

- Managed clusters must have internet access to reach `https://acme-v02.api.letsencrypt.org`
- For HTTP-01: a working Ingress controller with a publicly reachable IP
- For DNS-01: credentials for a supported DNS provider (Route53, CloudDNS, Azure DNS, etc.)

### ClusterIssuer (HTTP-01 Solver)

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: platform-team@example.com
    privateKeySecretRef:
      name: letsencrypt-prod-account-key
    solvers:
      - http01:
          ingress:
            ingressClassName: openshift-default
```

### ClusterIssuer (DNS-01 Solver — Route53 Example)

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-dns
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: platform-team@example.com
    privateKeySecretRef:
      name: letsencrypt-dns-account-key
    solvers:
      - dns01:
          route53:
            region: us-east-1
            accessKeyIDSecretRef:
              name: route53-credentials
              key: access-key-id
            secretAccessKeySecretRef:
              name: route53-credentials
              key: secret-access-key
```

For DNS-01, the credentials Secret must exist on each managed cluster. See fleet distribution below.

### Fleet Distribution

Distribute the ClusterIssuer (and any credential Secrets for DNS-01) via a ManifestWork or ManifestWorkReplicaSet from the hub:

```yaml
apiVersion: work.open-cluster-management.io/v1
kind: ManifestWork
metadata:
  name: cert-issuer-letsencrypt
  namespace: <cluster-name>
spec:
  workload:
    manifests:
      - apiVersion: cert-manager.io/v1
        kind: ClusterIssuer
        metadata:
          name: letsencrypt-prod
        spec:
          acme:
            server: https://acme-v02.api.letsencrypt.org/directory
            email: platform-team@example.com
            privateKeySecretRef:
              name: letsencrypt-prod-account-key
            solvers:
              - http01:
                  ingress:
                    ingressClassName: openshift-default
```

Alternatively, use an ACM Policy with `musthave` enforcement to ensure the ClusterIssuer exists on target clusters.

### Security Considerations

- Use the staging server (`https://acme-staging-v02.api.letsencrypt.org/directory`) for testing to avoid rate limits.
- The ACME account private key is auto-generated per cluster. It is not a shared secret.
- For DNS-01, the DNS provider credentials Secret is sensitive. Distribute it via ManifestWork (encrypted in transit) and scope RBAC carefully.
- HTTP-01 requires port 80 reachable from the internet. Ensure firewall rules allow this.

---

## 2. Internal CA (Self-Signed or Corporate CA)

### When to Use

- Internal services that do not need publicly trusted certificates
- Air-gapped or disconnected environments
- Organizations with an existing corporate CA chain

### Prerequisites

- A CA certificate and private key (PEM-encoded)
- For corporate CA: obtain the signing cert and key from your PKI team

### Create the CA Secret

On each managed cluster (or distribute via ManifestWork):

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: internal-ca-key-pair
  namespace: cert-manager
type: kubernetes.io/tls
data:
  tls.crt: <base64-encoded-CA-certificate>
  tls.key: <base64-encoded-CA-private-key>
```

### ClusterIssuer

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: internal-ca
spec:
  ca:
    secretName: internal-ca-key-pair
```

Note: The CA secret must be in the `cert-manager` namespace (or the namespace where cert-manager is installed). The ClusterIssuer references it by name.

### Self-Signed Bootstrap

If you do not have an existing CA, you can bootstrap one with cert-manager itself:

```yaml
# Step 1: Self-signed issuer to generate a CA cert
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: selfsigned-bootstrap
spec:
  selfSigned: {}
---
# Step 2: CA Certificate (issued by self-signed)
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: internal-ca-cert
  namespace: cert-manager
spec:
  isCA: true
  commonName: internal-ca
  duration: 87600h  # 10 years
  secretName: internal-ca-key-pair
  issuerRef:
    name: selfsigned-bootstrap
    kind: ClusterIssuer
---
# Step 3: CA issuer using the generated cert
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: internal-ca
spec:
  ca:
    secretName: internal-ca-key-pair
```

### Fleet Distribution

Distribute the CA Secret and ClusterIssuer together via ManifestWork:

```yaml
apiVersion: work.open-cluster-management.io/v1
kind: ManifestWork
metadata:
  name: cert-issuer-internal-ca
  namespace: <cluster-name>
spec:
  workload:
    manifests:
      - apiVersion: v1
        kind: Secret
        metadata:
          name: internal-ca-key-pair
          namespace: cert-manager
        type: kubernetes.io/tls
        data:
          tls.crt: <base64-encoded-CA-certificate>
          tls.key: <base64-encoded-CA-private-key>
      - apiVersion: cert-manager.io/v1
        kind: ClusterIssuer
        metadata:
          name: internal-ca
        spec:
          ca:
            secretName: internal-ca-key-pair
```

For the self-signed bootstrap pattern, distribute all three resources (ClusterIssuer, Certificate, CA ClusterIssuer).

### Security Considerations

- The CA private key is distributed to every managed cluster. If any cluster is compromised, the attacker can sign certificates for any domain in your CA scope.
- Consider using a dedicated intermediate CA for fleet distribution rather than your root CA.
- Rotate the CA certificate before expiry. Update the Secret on all clusters via ManifestWork.
- The self-signed bootstrap approach generates a unique CA per cluster. Use the corporate CA approach if you need a shared trust root.

---

## 3. HashiCorp Vault

### When to Use

- Centralized PKI management via Vault's PKI secrets engine
- Strict audit requirements for certificate issuance
- Organizations already running Vault as their secrets management platform

### Prerequisites

- A running Vault instance accessible from all managed clusters
- Vault PKI secrets engine enabled and configured with a CA
- An authentication method for cert-manager (Kubernetes ServiceAccount, AppRole, or Token)

### Vault PKI Setup (Vault CLI)

```bash
# Enable PKI engine
vault secrets enable pki

# Configure max TTL
vault secrets tune -max-lease-ttl=87600h pki

# Generate or import a root CA
vault write pki/root/generate/internal \
  common_name="vault-ca" ttl=87600h

# Create a role for cert-manager
vault write pki/roles/cert-manager-role \
  allowed_domains="example.com,svc.cluster.local" \
  allow_subdomains=true \
  max_ttl=8760h

# Create a policy for cert-manager
vault policy write cert-manager-policy - <<EOF
path "pki/sign/cert-manager-role" {
  capabilities = ["create", "update"]
}
path "pki/issue/cert-manager-role" {
  capabilities = ["create"]
}
EOF
```

### ClusterIssuer (Kubernetes Auth)

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: vault-issuer
spec:
  vault:
    server: https://vault.example.com:8200
    path: pki/sign/cert-manager-role
    caBundle: <base64-encoded-vault-CA-cert>
    auth:
      kubernetes:
        role: cert-manager
        mountPath: /v1/auth/kubernetes
        serviceAccountRef:
          name: cert-manager
```

### ClusterIssuer (AppRole Auth)

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: vault-issuer
spec:
  vault:
    server: https://vault.example.com:8200
    path: pki/sign/cert-manager-role
    caBundle: <base64-encoded-vault-CA-cert>
    auth:
      appRole:
        path: approle
        roleId: <vault-approle-role-id>
        secretRef:
          name: vault-approle-secret
          key: secret-id
```

The AppRole secret must exist on each managed cluster:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: vault-approle-secret
  namespace: cert-manager
type: Opaque
data:
  secret-id: <base64-encoded-secret-id>
```

### Fleet Distribution

Distribute the ClusterIssuer (and any auth Secrets) via ManifestWork:

```yaml
apiVersion: work.open-cluster-management.io/v1
kind: ManifestWork
metadata:
  name: cert-issuer-vault
  namespace: <cluster-name>
spec:
  workload:
    manifests:
      - apiVersion: v1
        kind: Secret
        metadata:
          name: vault-approle-secret
          namespace: cert-manager
        type: Opaque
        data:
          secret-id: <base64-encoded-secret-id>
      - apiVersion: cert-manager.io/v1
        kind: ClusterIssuer
        metadata:
          name: vault-issuer
        spec:
          vault:
            server: https://vault.example.com:8200
            path: pki/sign/cert-manager-role
            caBundle: <base64-encoded-vault-CA-cert>
            auth:
              appRole:
                path: approle
                roleId: <vault-approle-role-id>
                secretRef:
                  name: vault-approle-secret
                  key: secret-id
```

For Kubernetes auth, no additional Secret is needed per cluster if Vault is configured to trust each cluster's API server. Configure a separate Kubernetes auth mount per cluster in Vault.

### Security Considerations

- Kubernetes auth is preferred over AppRole for managed clusters — no shared secret to distribute.
- Each cluster needs a separate Kubernetes auth mount in Vault (different API server CA and endpoint).
- Scope Vault policies to specific PKI roles and domains. Avoid wildcard domain permissions.
- Ensure network connectivity from managed clusters to Vault. For private Vault, consider ClusterProxy or VPN.
- Rotate AppRole secret IDs periodically. Update the Secret on managed clusters via ManifestWork.

---

## Choosing an Issuer Pattern

| Pattern | Trust Level | Complexity | Fleet Suitability | Disconnected |
|---|---|---|---|---|
| Let's Encrypt (HTTP-01) | Public | Low | Good (no secrets to distribute) | No |
| Let's Encrypt (DNS-01) | Public | Medium | Medium (DNS credentials needed) | No |
| Internal CA | Private | Low | Medium (CA key distribution) | Yes |
| Self-Signed Bootstrap | Private (per-cluster) | Low | Good (no shared secrets) | Yes |
| Vault | Private | High | Good (centralized, audited) | Depends on Vault access |

For most fleet deployments, start with **Internal CA** (if you have an existing PKI) or **Let's Encrypt HTTP-01** (if clusters are publicly reachable). Use **Vault** when you need centralized audit and policy control over certificate issuance.
