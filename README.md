# Observability Setup
1. Update kubeconfig:
```
$ KUBECONFIG=~/.kube/config:~/Downloads/kube-config.yaml kubectl config view --flatten > /tmp/merged-config
$ mv /tmp/merged-config ~/.kube/config
$ chmod 600 ~/.kube/config
```

2. Install prometheus, loki and grafana.
```
$ helm version
$ helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
$ helm repo update
$ helm pull prometheus-community/kube-prometheus-stack --untar
$ kubectl create namespace obs
$ helm install prometheus ./kube-prometheus-stack -n obs
$ helm upgrade prometheus ./kube-prometheus-stack -n obs -f ./kube-prometheus-stack/values.yaml
$ kubectl get po -n obs
```

3. Install gateway api and expose the application, grafana using envoy gateway

```
$ helm pull oci://docker.io/envoyproxy/gateway-helm --untar
$ helm install envoygw ./gateway-helm -n envoy --create-namespace
$ kubectl apply -f quickstart.yaml
```

4. Setup application repo: https://github.com/gurjotkaur20/hello-world.git
    - Create helloworld application and its container locally.
    - Push the image in GHCR.
        ```
        $ echo "<YOUR_CLASSIC_PAT>" | docker login ghcr.io -u gurjotkaur20 --password-stdin
        $ docker tag helloworld:0.1 ghcr.io/gurjotkaur20/helloworld:0.1
        $ docker push ghcr.io/gurjotkaur20/helloworld:0.1
        ```
    - Create github action and push to the repo.
    - Create K8s manifests in the app repo
        ```
        $ kubectl create namespace app
        $ kubectl apply -f deployment.yaml 
        $ kubectl apply -f service.yaml
        ```

5. Collect app metrics on grafana.

    -   Expose application metrics on /metrics
    -   Name the container port
        ```yaml
        ports:
        - name: http
            containerPort: 3000
        ```
    -   Create a PodMonitor and Prometheus selects the PodMonitor
        ```yaml
        podMonitorSelector: {}
        podMonitorNamespaceSelector: {}
        ```
        → All PodMonitors across namespaces are eligible
    -   Verify Prometheus discovery. 
        -   Go to Prometheus UI → **Status → Targets**
        -   Look for: `podMonitor/app/hello-world/0`
        -   State should be: **UP**

    -   Verify query metrics in Prometheus
        ```
        http_requests_total
        http_request_duration_seconds
        ```

6. Setup ArgoCD
    - Install via helm
        ```
        $ helm repo add argocd https://argoproj.github.io/argo-helm
        $ helm repo update
        $ helm pull argocd/argo-cd --untar
        $ kubectl create namespace argocd
        $ helm install argocd ./argo-cd -n argocd
        $ kubectl get po -n argocd
        ```

    -   Disable HTTPS redirect via Envoy Gateway** (add to `argo-cd/values.yaml`):
        ```yaml
        server:
        extraArgs:
            - --insecure
        ```

    -   Then upgrade:
        ```
        $ helm upgrade argocd ./argo-cd -n argocd -f ./argo-cd/values.yaml
        ```
    -   Access ArgoCD UI via Envoy Gateway at: http://argocd.vapd.online:30237
    -   Create Application manifest argocd-application.yaml for GitOps deployment.

    **Issue faced:**    
    HTTPRoute was created accidentally in `default` namespace to reference `argocd-server` service in `argocd` namespace resulted in **HTTP 500 error**. Envoy Gateway blocked cross-namespace service references due to missing ReferenceGrant.   
    **Solution:** Moved HTTPRoute to `argocd` namespace where the service resides. Same-namespace references don't require ReferenceGrant.   
    **Recommendation:** Always create HTTPRoutes in the same namespace as the backend service to avoid security policy issues and complexity.
    
    **Verify deployment:**
    - `argocd app get hello-world` — Check application sync status
    - `kubectl get pods -n app` — Verify pods running
    - Access app at `http://app.vapd.online` — Test connectivity
    - ArgoCD auto-syncs on git changes within 3-5 minutes
    - Updated deployment.yaml with image tag manually and pushed the change, argocd detected automatically and synced the deployment again.

7. Automate CI/CD
    - CI workflow was already created in github action. Just added the below steps:
    
    **GitHub Actions steps:**
    - Build and push image with tags: `latest`, commit SHA, build number
    - Update deployment.yaml with new image tag:
        ```yaml
        - name: Update Deployment Image Tag
          run: |
            sed -i "s|image: ghcr.io/.*/hello-world:.*|image: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}|" k8s/deployment.yaml
        ```
    - Commit and push changes back to repository:
        ```yaml
        - name: Commit and Push Changes
          run: |
            git config user.email "github-actions[bot]@users.noreply.github.com"
            git config user.name "github-actions[bot]"
            git add k8s/deployment.yaml
            git commit -m "chore: update image tag to ${{ github.sha }}" || echo "No changes"
            git push origin main
        ```
    - ArgoCD detects git change → Auto-syncs → Deploys new version
---

## Later
1. Prepare helm chart for app
2. Enable https access on gateway
