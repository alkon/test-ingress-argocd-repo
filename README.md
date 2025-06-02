## 🧭 Manual Local Setup for Argo CD + Ingress + nip.io (Pre-Terraform)

This guide documents **manual configuration steps** to set up a local GitOps-ready Kubernetes cluster running Argo CD, exposed via an Ingress controller using a public-friendly `nip.io` domain.

The goal is to fully prepare the cluster so that later it can be automated via Terraform and GitHub Actions.

---

### 🚀 1. Install Argo CD via Helm
```bash
   helm repo add argo https://argoproj.github.io/argo-helm
   helm repo update

   helm upgrade --install argocd argo/argo-cd \
      --namespace argocd \
      --create-namespace
```
Verify: 
```bash
   kubectl get all -n argocd
```

### 🌐 2. Deploy Bitnami's NGINX Ingress Controller via Argo CD

- Manage the Ingress Controller declaratively via Argo CD itself, using a GitOps `Application` manifest `argocd-apps/nginx-oci-app.yaml`

- Note: Argo CD will take over the lifecycle of the Ingress Controller from this point forward 
- Reasons to delegate the lifecycle of the **Ingress Controller** to Argo CD (also in future Terraform version):
  - All configuration (chart version, ports, annotations) is Git-tracked and declarative
  - Argo CD can self-heal and re-apply the controller if it’s modified or deleted
  - Simplifies multi-environment consistency (dev/stage/prod)
  - Allows developers to modify Helm values without touching Terraform

```yaml
# Apply it once manually
kubectl apply -f argocd-apps/nginx-oci-app.yaml
```
### 4. Using `nip.io` to Expose Argo CD with Ingress

- We use `nip.io`— a public wildcard DNS service — to expose Argo CD via a domain like `argocd.192-168-65-3.nip.io` **without configuring DNS records manually**.

This enables GitHub Actions, browsers, and CLI tools to reach the `Argo CD API server` running inside the Kubernetes cluster.

- Step 1: Get the Internal Node IP: Take note of the INTERNAL-IP column value (e.g., 192.168.65.3).
```bash
   # Verify cluster info
   kubectl cluster-info
   kubectl get nodes -o wide
```
   
- Step 2: Confirm Ingress Controller NodePort Ports
  - Note: in output, like `nginx-ingress-nginx-ingress-controller   NodePort   ...   80:30782/TCP, 443:32165/TCP`
  look for the **HTTP** NodePort (e.g., 30782).
```bash
   kubectl get svc -n ingress-nginx-bitnami
```
  

- Step 3: Construct a nip.io Hostname
  - Transform the internal IP into a DNS-safe format: this will resolve to the Internal Node IP without needing to set up DNS
  ```text
     192.168.65.3 → 192-168-65-3.nip.io
  ```

