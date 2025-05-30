# GitHub Actions Self-Hosted Runner Deployment Troubleshooting

This README summarizes the common pitfalls and their solutions encountered during the setup and troubleshooting of a GitHub Actions workflow deploying Kubernetes resources (ArgoCD Ingress, ArgoCD Application, Nginx manifests) to a Kubernetes cluster via a self-hosted runner.

## The Journey of Troubleshooting: A Sequential Summary

Our goal was to automate the deployment of Kubernetes resources using a GitHub Actions workflow triggered by changes to specific YAML files in the repository. The workflow runs on a self-hosted runner and uses `kubectl` to interact with a local Kubernetes cluster (Docker Desktop).

### 1. Initial "File Does Not Exist" Error

**Problem:** The workflow failed with errors like `"argocd-ingress/argocd-ingress-http.yaml" does not exist`.
**Root Cause:**
    * **Incorrect `kubectl apply -f` path:** The `run` command in the workflow specified a path that didn't match the actual file location in the repository or contained typos (e.g., expecting `argocd-ingress-http-modified.yaml` when the file was `argocd-ingress-http.yaml`).
    * **File not tracked by Git (`.gitignore`):** Essential YAML files were accidentally ignored by Git and thus not present in the repository when the runner checked out the code.
**Solution:**
    * **Corrected file paths** in the `kubectl apply -f` commands within the workflow.
    * **Verified `.gitignore`** to ensure all necessary configuration and manifest files were tracked and pushed to the repository.

### 2. Workflow Not Triggering / "Empty Push Did Nothing"

**Problem:** After correcting file paths, the workflow still wouldn't run, even with an empty commit.
**Root Cause:**
    * **Syntactic errors in Workflow YAML:** Critical syntax errors (e.g., malformed `run` commands, incorrect indentation) within the `deploy-argocd-app.yaml` file on GitHub.com prevented GitHub Actions from parsing and registering the workflow.
**Solution:**
    * Thoroughly **reviewed and corrected all YAML syntax issues** in the workflow file, particularly ensuring `kubectl` commands had correct flags (e.g., `kubectl apply -f` instead of `kubectl apply f`). Pushing the corrected workflow file allowed it to be recognized.

### 3. `kubectl` OpenAPI Validation Error (Connection Refused)

**Problem:** The workflow started running, but `kubectl apply` failed with `failed to download openapi: Get "http://localhost:8080/openapi/v2": dial tcp [::1]:8080: connect: connection refused`.
**Root Cause:**
    * **Misconfigured `kubeconfig` or network accessibility:** `kubectl` was trying to fetch OpenAPI schemas for validation from `localhost:8080`, despite the `kubeconfig` pointing to `https://127.0.0.1:6443` (Docker Desktop). This indicated a connectivity issue for the validation endpoint.
**Solution:**
    * **Added `--validate=false`** to the `kubectl apply -f` commands in the workflow. This bypassed the problematic OpenAPI validation step, allowing `kubectl` to proceed with applying resources. (Note: The fundamental `kubeconfig` setup and runner connectivity were crucial here, which were confirmed in parallel steps).

### 4. `kubectl wait` Hanging Indefinitely

**Problem:** After `kubectl apply` appeared to run, the workflow hung at `kubectl wait --for=condition=Ready application/nginx-test-app` (or similar for Helm/OCI apps).
**Root Cause:**
    * **Incorrect `kubectl wait` condition for ArgoCD Application:** ArgoCD `Application` custom resources report their status via `status.health.status` (e.g., `Healthy`) and `status.sync.status` (e.g., `Synced`), not a generic `Ready` condition. The command was waiting for a non-existent condition.
    * **Timing/Propagation Delay:** Even with the correct condition, it might take a minute or more for Kubernetes to pull images, start containers, and for ArgoCD to update its application status to `Healthy` after syncing.
**Solution:**
    * **Modified the `kubectl wait` command** to use `jsonpath` to explicitly check the `status.health.status` field for `Healthy`:
        `kubectl wait --for=jsonpath='{.status.health.status}'=Healthy application/nginx-test-app -n argocd --timeout=5m`
    * Acknowledged that some delay in the `wait` step is normal due to resource provisioning and status propagation.

### 5. Accessing ArgoCD UI & Ingress Configuration

**Clarification:** The `argocd-ingress-http.yaml` resource is primarily for exposing the **ArgoCD web UI and API to external clients** (e.g., for browser access). **It is not required for the workflow's `kubectl` operations or ArgoCD's core GitOps functionality.**

**Why the Ingress is not required for core operations:**

* **For the workflow's `kubectl` operations:**
    * The `kubectl` commands executed by your self-hosted GitHub Actions runner communicate directly with the **Kubernetes API server**.
    * This communication happens via the `kubeconfig` file (which contains authentication details and the API server endpoint, e.g., `https://127.0.0.1:6443` for Docker Desktop).
    * The Ingress resource's purpose is to route external HTTP/S traffic *into* the cluster to a specific service (ArgoCD UI/API). It has no role in `kubectl`'s administrative interactions with the cluster's control plane. `kubectl` is a direct client of the Kubernetes API, not a web client that goes through an Ingress.

* **For ArgoCD's core GitOps functionality (Git-to-Cluster Sync):**
    * ArgoCD's primary function is to continuously observe your Git repository (e.g., GitHub.com) for changes to your application manifests. It pulls these changes using standard Git protocols (HTTPS or SSH).
    * It then applies these manifests to your Kubernetes cluster by making calls directly to the **Kubernetes API server** (using its own internal service account credentials).
    * These core synchronization processes happen entirely within the cluster's network, or by ArgoCD reaching *out* to Git. They are completely independent of whether the ArgoCD UI or API is exposed externally via an Ingress. The Ingress merely provides a user-friendly entry point for *you* to interact with ArgoCD, not for ArgoCD to perform its duties.

**Accessing the UI (without `kubectl port-forward`):**
    * **Ingress Configuration:** Ensure the Ingress resource is deployed and configured with a hostname (e.g., `argocd.yourdomain.com` or `argocd.127.0.0.1.nip.io`).
    * **Local DNS Resolution (`/etc/hosts` / `nip.io`):**
        * For custom hostnames (e.g., `argocd.yourdomain.com`), you need to manually add an entry to your machine's `hosts` file (e.g., `127.0.0.1    argocd.yourdomain.com`) to map the Ingress hostname to your local Kubernetes Ingress controller's IP.
        * Alternatively, using a `nip.io` hostname (e.g., `argocd.127.0.0.1.nip.io`) eliminates the need for `/etc/hosts` modifications, as `nip.io` automatically resolves the hostname to the embedded IP address.
    * **Retrieving Admin Password:** The initial ArgoCD admin password is required for login and can be retrieved using:
        `kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 --decode; echo`
    * Navigate to the configured hostname in your browser (e.g., `http://argocd.127.0.0.1.nip.io`) and log in with username `admin` and the retrieved password.

**Explanation of Nginx Ingress Annotations (`nginx.ingress.kubernetes.io`):**
These annotations are crucial for the specific setup of an **HTTP Ingress** (port 80) pointing to an **HTTPS-expecting backend** (ArgoCD server) when using the **Nginx Ingress Controller**:
    * **`nginx.ingress.kubernetes.io/force-ssl-redirect: "false"`:** This annotation tells the Nginx Ingress Controller *not* to automatically redirect incoming HTTP requests to HTTPS. This is necessary in a local development setup where you likely don't have proper TLS certificates configured for the Ingress, preventing browser errors due to unconfigured HTTPS redirects.
    * **`nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"`:** This annotation instructs the Nginx Ingress Controller to use HTTPS when it communicates with the backend `argocd-server` service, even if the external client (your browser) is connecting via HTTP. This bridges the protocol gap, as the `argocd-server` itself serves its UI and API internally over HTTPS.

**Workarounds (if Ingress is not desired):**
    * **Remove the Ingress step** from your workflow.
    * Use `kubectl port-forward svc/argocd-server -n argocd 8080:80` on your local machine to temporarily access the ArgoCD UI securely via `http://localhost:8080`.

---

By systematically addressing these issues, the GitHub Actions workflow was successfully configured to deploy resources to the Kubernetes cluster, supporting raw YAML, Helm chart, and OCI Helm chart deployments.