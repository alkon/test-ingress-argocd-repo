# K3d, Argo CD, and NGINX Ingress Controller Clean Installation Guide

This guide provides comprehensive steps for a clean installation of a local Kubernetes cluster using K3d, deploying Argo CD for GitOps, and setting up the Bitnami NGINX Ingress Controller to expose the Argo CD UI via HTTPS with nip.io DNS.

**Important Pre-requisites:**

* Docker Desktop installed and running.
* `kubectl` installed.
* `k3d` installed.
* `openssl` installed.

## Phase 1: Clean Up and Recreate K3d Cluster

This phase ensures a fresh environment by restarting Docker and recreating your K3d cluster. This is crucial for resolving underlying network issues that can occur in local Docker/K3d setups.

1.  **Quit Docker Desktop completely.**
    * On macOS, click the Docker whale icon in your menu bar (top right) and select "Quit Docker Desktop."
    * On Windows, right-click the Docker icon in the system tray and select "Quit Docker Desktop."
2.  **Restart Docker Desktop.** Launch Docker Desktop again from your Applications folder (macOS) or Start Menu (Windows). Wait for it to show that it's fully running (the whale icon should be steady).
3.  **Delete your existing K3d cluster:**

    ```bash
    k3d cluster delete argocd-cluster
    ```

4.  **Recreate your K3d cluster:** This command ensures the cluster is created with a specific K3s image version and correctly maps ports 80 and 443 to the load balancer for external access.

    ```bash
    k3d cluster create argocd-cluster \
      --image rancher/k3s:v1.31.5-k3s1 \
      --api-port 6550 \
      --servers 1 \
      --agents 0 \
      --port "80:80@loadbalancer" \
      --port "443:443@loadbalancer"
      --k3s-arg "--disable=traefik@server:0" # <--- ADD THIS LINE!
    ```

5.  Wait for the cluster to be fully ready.

## Phase 2: Install Argo CD

This phase deploys the core Argo CD components into your newly created K3d cluster.

1.  **Create the `argocd` namespace:**

    ```bash
    kubectl create namespace argocd
    ```

2.  **Apply the official Argo CD installation manifests:**

    ```bash
    kubectl apply -n argocd -f [https://raw.githubusercontent.com/argoproj/argo-cd/v3.0.5/manifests/install.yaml](https://raw.githubusercontent.com/argoproj/argo-cd/v3.0.5/manifests/install.yaml)
    ```

3.  **Wait for all Argo CD pods to become `Running` in the `argocd` namespace before proceeding:**

    ```bash
    kubectl get pods -n argocd -w
    ```

### Initial Argo CD Access (for verification/debugging)

While we'll use Ingress for long-term access, `kubectl port-forward` is useful for initial verification and debugging.

1.  **Access Argo CD UI (via port-forward):**

    ```bash
    kubectl port-forward svc/argocd-server -n argocd 8080:443
    ```

    Then open `https://localhost:8080` in your browser (you'll need to bypass security warnings).
2.  **Retrieve the initial admin password:**

    ```bash
    kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath="{.data.password}" | base64 -d
    ```

## Phase 3: Deploy NGINX Ingress Controller (via Argo CD)

This phase uses Argo CD to declaratively deploy the Bitnami NGINX Ingress Controller.

1.  **Create the Argo CD Application manifest for the Ingress Controller:**
    Save the following content as `argocd-apps/nginx-ingress-controller-app.yaml` in your Git repository. This manifest tells Argo CD to deploy the `nginx-ingress-controller` Helm chart.

```yaml
# argocd-apps/nginx-ingress-controller-app.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
name: nginx-ingress-controller-app # A distinct name to indicate it's the Ingress Controller
namespace: argocd # Argo CD application definition lives in the argocd namespace
spec:
project: default
source:
repoURL: registry-1.docker.io/bitnamicharts # IMPORTANT: OCI prefix for OCI Helm charts
chart: nginx-ingress-controller # CORRECT chart name for the Ingress Controller
targetRevision: 11.6.22 # IMPORTANT: Use a current and stable version for this chart. Check Artifact Hub!
helm:
  releaseName: nginx-ingress-controller-release # A specific release name for Helm
  values: |
    controller: # Ingress Controller-specific values are nested under 'controller'
      replicaCount: 1
      service:
        type: LoadBalancer # ESSENTIAL for K3d to expose via host ports
        externalTrafficPolicy: "Local"
        ports:
          http: 80
          https: 443
      ingressClassResource: # Ensure the IngressClass is created and named correctly
        name: nginx
        enabled: true
        default: false # Set to true if you want it to be the default for all Ingresses
  parameters:
    - name: controller.service.externalTrafficPolicy
      value: "Local" # Ensure this is passed as a string  externalTrafficPolicy: Local
destination:
server: https://kubernetes.default.svc
namespace: ingress-nginx # Common and clear namespace for the Ingress Controller
syncPolicy:
automated:
  prune: true
  selfHeal: true
syncOptions:
  - CreateNamespace=true # Automatically create the target namespace if it doesn't exist
  # - ApplyOutOfSyncOnly=true # Consider this for advanced scenarios to reduce reconciliation noise
```

2.  **Apply the Argo CD Application manifest:**

    ```bash
    kubectl apply -n argocd -f argocd-apps/nginx-ingress-controller-app.yaml
    ```

3.  **Monitor Argo CD to ensure it deploys the Ingress Controller:**Wait for the Ingress Controller pods to become `Running` in the `ingress-nginx` namespace:

    ```bash
    kubectl get pods -n ingress-nginx -w
    ```

## Phase 4: Deploy Argo CD UI Ingress

This phase configures the NGINX Ingress Controller to expose the Argo CD UI via HTTPS using a nip.io hostname and a self-signed TLS certificate.

1.  **Get the NEW K3d LoadBalancer IP Address:**
    After recreating the K3d cluster, its IP address will likely have changed. This is the IP you'll use for your nip.io hostname.

    ```bash
    docker inspect k3d-argocd-cluster-serverlb | grep IPAddress
    ```

    Note down the IP (e.g., `172.18.0.X`). This will be `<YOUR_K3D_LB_IP>`.
2.  **Generate a Self-Signed TLS Certificate and Key:**
    These commands create the necessary files for HTTPS. Replace `<YOUR_K3D_LB_IP_DASHED>` with your actual K3d LoadBalancer IP, with dots replaced by hyphens (e.g., `172-18-0-3` if the IP is `172.18.0.3`).

```bash
   openssl genrsa -out tls.key 2048
```
```bash
   openssl req -new -key tls.key -out tls.csr -subj "/CN=argocd.<YOUR_K3D_LB_IP_DASHED>.nip.io/O=MyArgoCDOrg"
```
```bash
   openssl x509 -req -days 365 -in tls.csr -signkey tls.key -out tls.crt
```

3.  **Create a Kubernetes TLS Secret:**

    ```bash
    kubectl create secret tls argocd-tls --cert=tls.crt --key=tls.key -n argocd
    ```

4.  **Update and Apply your `argocd-ingress.yaml`:**VERY IMPORTANT: Update both `host:` and `tls.hosts:` fields in this file with the NEW K3d IP you obtained in step 1 of this phase.

    ```yaml
    # argocd-ingress/argocd-ingress.yaml
    apiVersion: networking.k8s.io/v1
    kind: Ingress
    metadata:
      name: argocd-ingress
      namespace: argocd
      annotations:
        # This annotation is less critical now that TLS is configured
        # nginx.ingress.kubernetes.io/ssl-redirect: "false"
    spec:
      ingressClassName: nginx
      rules:
        - host: argocd.<YOUR_K3D_LB_IP_DASHED>.nip.io # <--- UPDATE THIS IP!
          http:
            paths:
              - path: /
                pathType: Prefix
                backend:
                  service:
                    name: argocd-server
                    port:
                      number: 443 # Ensure this is 443 for HTTPS backend
      tls: # This block enables HTTPS for the Ingress
        - hosts:
            - argocd.<YOUR_K3D_LB_IP_DASHED>.nip.io # <--- UPDATE THIS IP!
          secretName: argocd-tls # Name of the TLS secret created above
    ```

5.  **Apply the updated Ingress manifest:**

    ```bash
    kubectl apply -f argocd-ingress/argocd-ingress.yaml
    ```

## Phase 5: Final Connectivity Test

Now, let's verify that you can access the Argo CD UI via the Ingress.

1.  **Crucial `nc` Test from your Host Machine:**
    This test verifies that your host can connect to the K3d LoadBalancer IP on port 443. This must succeed for browser access to work.

    ```bash
    nc -vz <YOUR_K3D_LB_IP> 443
    ```

    Expected Output: `succeeded!` (e.g., `Connection to 172.18.0.3 443 port [tcp/https] succeeded!`)If it times out, the problem is still a host-level network block (e.g., antivirus, VPN, or a deeper Docker networking issue).
2.  **Verify Ingress Status:**

    ```bash
    kubectl get ingress -n argocd argocd-ingress
    ```

    Confirm that the `HOSTS` and `ADDRESS` columns correctly reflect your nip.io hostname and the K3d LoadBalancer IP.
3.  **Access Argo CD UI in your browser:**
    Open your web browser and navigate to:

    ```
    https://argocd.<YOUR_K3D_LB_IP_DASHED>.nip.io
    ```

    Remember to bypass the browser's security warning for the self-signed certificate. This is expected and necessary for local testing with self-signed certs.
    If all steps are followed correctly, you should now see the Argo CD login page!