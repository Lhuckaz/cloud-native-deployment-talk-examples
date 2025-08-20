# Exemplos usados na demonstração da talk

[Slides](https://docs.google.com/presentation/d/18bHT_kW2bx_kLBFqpBLuqzlVUUoI9RXreCrnmqKm2Fk/edit?usp=sharing)

## Deployment Steps for Docker Desktop Kubernetes with Argo CD

This section outlines the steps to deploy the applications in this repository to a Docker Desktop Kubernetes cluster using Argo CD.

### Prerequisites

*   Docker Desktop with Kubernetes enabled
*   `kubectl` configured to connect to your Docker Desktop Kubernetes cluster
*   Argo CD installed in your cluster (e.g., in the `argocd` namespace)
*   `argocd` CLI installed and configured (logged in)
*   `git` installed and configured to push to your repository

### Step-by-Step Deployment

1.  **Configure Argo CD ApplicationSet for Git Source:**
    The `blue-green/simple/blue-green.yml` file is an Argo CD ApplicationSet that was initially configured to pull a Helm chart from a ChartMuseum instance. We need to modify it to pull directly from your Git repository.

    *   **Modify `blue-green/simple/blue-green.yml`:**
        Replace the `source` section to point to your Git repository.
        ```yaml
        # Original source section (example)
        # source:
        #   repoURL: 'http://chartmuseum.chartmuseum.svc.cluster.local:8080'
        #   chart: sample
        #   targetRevision: 0.1.0

        # New source section (replace with your repository URL)
        source:
          repoURL: 'https://github.com/Lhuckaz/cloud-native-deployment-talk-examples' # Your Git repository URL
          path: helm/sample
          targetRevision: HEAD # Or your desired branch/tag
        ```

    *   **Apply the ApplicationSet:**
        ```bash
        kubectl apply -f blue-green/simple/blue-green.yml
        ```

2.  **Create Argo CD Project:**
    The ApplicationSet attempts to deploy the application into an Argo CD project named `chip`. This project needs to be created if it doesn't exist.

    *   **Create the `chip` project:**
        ```bash
        argocd proj create chip --description "Project for the chip application" --src "*" --dest "*,*" --allow-cluster-resource "*" --allow-namespaced-resource "*" --grpc-web
        ```

    *   **Refresh the ApplicationSet:**
        After creating the project, you might need to force the ApplicationSet controller to re-evaluate.
        ```bash
        kubectl annotate applicationset chip -n argocd gemini-cli-refresh=1 --overwrite
        ```

3.  **Fix Helm Chart ServiceAccount Template:**
    An issue was identified in the `helm/sample/templates/service-account.yaml` where an empty `iam` value in `values.yaml` caused a `nil` value for the `eks.amazonaws.com/role-arn` annotation, leading to a `ComparisonError`.

    *   **Modify `helm/sample/templates/service-account.yaml`:**
        Update the file to conditionally render the `annotations` block:
        ```yaml
        apiVersion: v1
        kind: ServiceAccount
        metadata:
          {{- if .Values.app.iam }}
          annotations:
            eks.amazonaws.com/role-arn: {{ .Values.app.iam }}
          {{- end }}
          name: {{ .Values.app.name }}
          namespace: {{ .Values.app.namespace }}
        ```

    *   **Commit and Push the Change:**
        Argo CD pulls from your remote Git repository, so you must commit and push this change.
        ```bash
        git add helm/sample/templates/service-account.yaml
        git commit -m "Fix ServiceAccount template for empty IAM role"
        git push
        ```

4.  **Install Missing CRDs (Istio and Argo Rollouts):**
    The application requires Custom Resource Definitions (CRDs) for Istio and Argo Rollouts.

    *   **Install Istio CRDs:**
        First, download `istioctl`:
        ```bash
        curl -L https://istio.io/downloadIstio | sh -
        ```
        Then, install the minimal Istio components (which include CRDs):
        ```bash
        ./istio-1.27.0/bin/istioctl install --set profile=minimal -y
        ```

    *   **Install Argo Rollouts CRDs:**
        Create the namespace for Argo Rollouts:
        ```bash
        kubectl create namespace argo-rollouts
        ```
        Apply the Argo Rollouts installation manifest (includes CRDs and controller):
        ```bash
        kubectl apply -n argo-rollouts -f https://github.com/argoproj/argo-rollouts/releases/latest/download/install.yaml
        ```

5.  **Install Istio Ingress Gateway:**
    The minimal Istio profile does not include the Ingress Gateway by default. This gateway is necessary to expose your application externally.

    *   **Install the Ingress Gateway:**
        ```bash
        ./istio-1.27.0/bin/istioctl install --set profile=minimal --set components.ingressGateways[0].name=istio-ingressgateway --set components.ingressGateways[0].namespace=istio-system -y
        ```

6.  **Access the Application:**
    The application is exposed via the Istio Ingress Gateway. You'll need to configure your local machine to resolve the application's hostname and then access it through the gateway's exposed port.

    *   **Get the Ingress Gateway IP and Port:**
        ```bash
        kubectl get svc istio-ingressgateway -n istio-system
        ```
        Look for the `EXTERNAL-IP` (which might be `localhost` on Docker Desktop) and the port mapped to `80/TCP` (e.g., `31547`).

    *   **Modify your Hosts File:**
        Add an entry to your hosts file to map the application's hostname (`chip.homelab.msfidelis.com.br`) to the Ingress Gateway's IP address (e.g., `127.0.0.1`).

        **On Linux/macOS:**
        Edit `/etc/hosts` and add the following line:
        ```
        127.0.0.1 chip.homelab.msfidelis.com.br
        ```

        **On Windows:**
        Edit `C:\Windows\System32\drivers\etc\hosts` and add the following line:
        ```
        127.0.0.1 chip.homelab.msfidelis.com.br
        ```

    *   **Access the Application:**
        After saving your hosts file, you can access your application in your web browser at:
        `http://chip.homelab.msfidelis.com.br:<NODE_PORT>` (replace `<NODE_PORT>` with the port you found in the `kubectl get svc` command, e.g., `31547`).

### Verifying Deployment

After completing these steps, Argo CD should automatically sync the `chip` application. You can monitor its status via the Argo CD UI or by using the `argocd` CLI:

```bash
argocd app get chip --grpc-web --refresh
```