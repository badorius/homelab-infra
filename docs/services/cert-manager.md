# cert-manager

Automatic TLS certificate management for all cluster services.

## Access

| Property | Value |
|----------|-------|
| Namespace | cert-manager |
| Chart | jetstack/cert-manager v1.14.4 |
| Managed by | Manual bootstrap (`kubectl apply -k kubernetes/infra/cert-manager/`) |

## How It Works

```
selfsigned-issuer (ClusterIssuer)
        │
        └─ issues ──▶ homelab-ca (Certificate, stored in root-secret)
                              │
                              └─ homelab-ca (ClusterIssuer)
                                        │
                                        └─ signs ──▶ *.home service certificates
                                                    (auto-created when ingress has annotation)
```

## Issuing a Certificate

Add these to any Ingress:

```yaml
metadata:
  annotations:
    cert-manager.io/cluster-issuer: homelab-ca
spec:
  ingressClassName: traefik
  tls:
    - hosts:
        - myservice.home
      secretName: myservice-tls
```

cert-manager detects the annotation and creates a `Certificate` resource. Within ~30 seconds, `myservice-tls` Secret appears in the namespace with the signed cert.

## Common Operations

```bash
# Check all certificates
kubectl get certificates -A

# Check why a certificate is not Ready
kubectl describe certificate <name> -n <namespace>
kubectl get certificaterequest -A
kubectl describe certificaterequest <name> -n <namespace>

# Force reissue a certificate
kubectl delete certificate <name> -n <namespace>
# cert-manager will recreate it immediately

# Export the root CA certificate (import into browser/OS)
kubectl get secret root-secret -n cert-manager \
  -o jsonpath='{.data.ca\.crt}' | base64 -d > homelab-root-ca.crt
```

## Trusting the Root CA

### Arch Linux (physical hosts)
```bash
sudo cp homelab-root-ca.crt /etc/ca-certificates/trust-source/anchors/
sudo update-ca-trust
```

### Ubuntu/Debian (VMs)
```bash
sudo cp homelab-root-ca.crt /usr/local/share/ca-certificates/
sudo update-ca-certificates
```

### Docker (for pushing to Harbor)
```bash
sudo mkdir -p /etc/docker/certs.d/registry.home
sudo cp homelab-root-ca.crt /etc/docker/certs.d/registry.home/ca.crt
sudo systemctl restart docker
```

### Browser
Settings → Privacy → Certificates → Import → select `homelab-root-ca.crt` → trust for websites.

## Distribute CA via Ansible

```bash
# Copy CA to Ansible files directory
cp homelab-root-ca.crt ansible/roles/common/files/homelab-root-ca.crt

# Distribute to all hosts and VMs
ansible-playbook playbooks/setup_hosts.yml --tags ca_trust
ansible-playbook playbooks/setup_k3s.yml --tags ca_trust
```
