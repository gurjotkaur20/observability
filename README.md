# Observability Setup
## 1. End to end setup over http
### 1.1 Update kubeconfig:
```
$ KUBECONFIG=~/.kube/config:~/Downloads/kube-config.yaml kubectl config view --flatten > /tmp/merged-config
$ mv /tmp/merged-config ~/.kube/config
$ chmod 600 ~/.kube/config
```

### 1.2 Install prometheus and grafana.
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

### 1.3 Install gateway api and expose grafana

```
$ helm pull oci://docker.io/envoyproxy/gateway-helm --untar
$ helm install envoygw ./gateway-helm -n envoy --create-namespace
$ kubectl apply -f quickstart.yaml
```

### 1.4 Setup application 
Repo: https://github.com/gurjotkaur20/hello-world.git   
-   Create helloworld application and its container locally.   
-   Push the image in GHCR.
    ```
    $ echo "<YOUR_CLASSIC_PAT>" | docker login ghcr.io -u gurjotkaur20 --password-stdin
    $ docker tag helloworld:0.1 ghcr.io/gurjotkaur20/helloworld:0.1
    $ docker push ghcr.io/gurjotkaur20/helloworld:0.1
    ```
-   Create github action and push to the repo.   
-   Create K8s manifests in the app repo   
    ```
    $ kubectl create namespace app
    $ kubectl apply -f deployment.yaml 
    $ kubectl apply -f service.yaml
    ```

### 1.5 Collect app metrics on grafana.

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

### 1.6 Setup ArgoCD
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

### 1.7 Automate CI/CD
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

## 2. Enable HTTPS

### 2.1 Install cert-manager
Cert-manager automatically provisions and manages TLS certificates for your domains.

```bash
# Add cert-manager Helm repository
$ helm repo add jetstack https://charts.jetstack.io
$ helm repo update

# Pull the cert-manager chart
$ helm pull jetstack/cert-manager --untar

# Create namespace and install cert-manager with CRDs
$ kubectl create namespace cert-manager
$ helm install cert-manager ./cert-manager -n cert-manager --set installCRDs=true

# Verify installation
$ kubectl get po -n cert-manager
```

### 2.2 Create ClusterIssuer for Let's Encrypt
Cert-manager automatically provisions and manages TLS certificates for your domains.

```bash
$ kubectl apply -f - <<EOF
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: gurjotkaur20@gmail.com  # Replace with your email address
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
    - http01:
        gatewayHTTPRoute:
          parentRefs:
          - name: eg-gateway
            namespace: obs
EOF
```

**Verify ClusterIssuer creation:**

```bash
# List all ClusterIssuers
$ kubectl get clusterissuer
```

### 2.3 Update Envoy Gateway to support HTTPS
Modify the gateway configuration to listen on port 443 with TLS termination.

Edit `quickstart.yaml` and update the Gateway resource:

```yaml
listeners:
    - name: http
        protocol: HTTP
        port: 80
        allowedRoutes:
        namespaces:
            from: All  
    - name: https
        protocol: HTTPS
        port: 443
        allowedRoutes:
        namespaces:
            from: All
        tls:
        mode: Terminate
        certificateRefs:
        - name: vapd-online-cert-secret
            namespace: default                   
```

### 2.4 Create Certificate for your domain
Create a Certificate resource to request TLS certificate from Let's Encrypt.

```bash
$ kubectl apply -f - <<EOF
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: vapd-online-cert
  namespace: default
spec:
  secretName: vapd-online-cert-secret
  commonName: vapd.online
  dnsNames:
  - vapd.online
  - "*.vapd.online"
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
EOF
```

**Note:** The wildcard `*.vapd.online` covers all subdomains (grafana, app, argocd), so specific domain names like `app.vapd.online` should not be listed separately.

### 2.5 Update HTTPRoutes for HTTPS
HTTPRoutes automatically work with both HTTP and HTTPS once the Gateway is configured. No changes needed to your existing HTTPRoute resources.

However, you can optionally redirect HTTP to HTTPS by adding a redirect filter (requires Envoy Gateway v1.1+):

```yaml
apiVersion: gateway.networking.k8s.io/v1beta1
kind: HTTPRoute
metadata:
  name: redirect-http-to-https
  namespace: default
spec:
  parentRefs:
  - name: my-gateway
  hostnames:
  - "*.vapd.online"
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /
    filters:
    - type: RequestRedirect
      requestRedirect:
        scheme: https
        statusCode: 301
```

### 2.6 Verify HTTPS Setup
```bash
# Check certificate status
$ kubectl get certificate -n default
$ kubectl describe certificate vapd-online-cert -n default

# Check if certificate secret was created
$ kubectl get secret vapd-online-cert-secret -n default

# Test access
$ curl -v https://grafana.vapd.online
$ curl -v https://app.vapd.online
$ curl -v https://argocd.vapd.online
```

### 2.7 Troubleshooting
**Certificate not issuing?**
- Check cert-manager logs: `kubectl logs -n cert-manager deploy/cert-manager`
- Check Certificate status: `kubectl describe certificate vapd-online-cert -n default`
- Ensure DNS is pointing to your Envoy Gateway IP/hostname
- Verify ACME challenge can reach your gateway on port 80

**HTTPS still not working?**
- Confirm Gateway listeners are updated: `kubectl get gateway -o yaml`
- Check if TLS secret exists: `kubectl get secret vapd-online-cert-secret`
- Verify DNS propagation: `nslookup grafana.vapd.online`
- Check Envoy Gateway logs: `kubectl logs -n envoy deploy/envoy-gateway`

---

## 2.8 Issues Faced and Fixed

The important part is that **there were actually two separate problems**, and we solved them independently.

### Final architecture

```
                         Let's Encrypt
                              |
                              | HTTP-01 :80
                              v
                    15.235.203.23:80
                              |
                    iptables DNAT/SNAT
                              |
                              v
                    Envoy NodePort :30237
                              |
                              v
                     Envoy Gateway
                              |
                    cert-manager solver
                       HTTPRoute
                              |
                              v
                    HTTP-01 challenge
```

### What we fixed

**1. cert-manager Gateway API was initially disabled**

The challenge initially failed with:

```
gateway api is not enabled
```

We enabled it in cert-manager:

```
config:
  apiVersion: controller.config.cert-manager.io/v1alpha1
  kind: ControllerConfiguration
  gatewayAPI:
    enabled: true
```

After that:

```
Presented: true
```

So cert-manager successfully created the temporary Gateway API HTTPRoutes.

---

**2. Port 80 was not publicly exposed**

Your Envoy service was:

```
NodePort
80:30237
```

So:

```
15.235.203.23:80    ❌
15.235.203.23:30237  ✅
```

We needed:

```
public :80 → NodePort :30237
```

We added:

```
sudo iptables -t nat -A PREROUTING \
  -i ens3 \
  -p tcp \
  -d 15.235.203.23 \
  --dport 80 \
  -j REDIRECT --to-port 30237
```

This made external traffic work:

```
Laptop
  ↓
15.235.203.23:80
  ↓
30237
  ↓
Envoy
```

---

**3. But cert-manager itself still couldn't reach port 80**

This was the tricky part.

External:

```
Laptop → 15.235.203.23:80 → Envoy    ✅
```

But cert-manager:

```
Pod → 15.235.203.23:80 → ❌
```

We proved that the pod could directly reach the NodePort:

```
Pod → 15.235.203.23:30237 → Envoy    ✅
```

but not port 80.

So we added a pod-CIDR-specific **DNAT**:

```
sudo iptables -t nat -A PREROUTING \
  -s 10.233.64.0/18 \
  -p tcp \
  -d 15.235.203.23 \
  --dport 80 \
  -j DNAT \
  --to-destination 15.235.203.23:30237
```

and **MASQUERADE/SNAT**:

```
sudo iptables -t nat -A POSTROUTING \
  -s 10.233.64.0/18 \
  -d 15.235.203.23 \
  -p tcp \
  --dport 30237 \
  -j MASQUERADE
```

This solved the Kubernetes pod → public-IP hairpin:

```
cert-manager Pod
10.233.123.x
      |
      | :80
      v
15.235.203.23
      |
      | DNAT + SNAT
      v
30237
      |
      v
Envoy
```

We verified it with:

```
Established connection to 15.235.203.23 port 80
HTTP/1.1 404 Not Found
```

The `404` was actually **good** at this stage—it proved the request reached Envoy. `/test` wasn't a real ACME token, so 404 was expected.

---

**4. DNS issue**

We also discovered:

```
vapd.online → 3.33.130.190 / 15.197.148.33
```

while your Kubernetes node was:

```
15.235.203.23
```

Since you didn't need the apex domain, we removed:

```
- vapd.online
```

from the certificate and kept:

```
grafana.vapd.online
app.vapd.online
argocd.vapd.online
```

---

**Final result**

Once the pod could reach:

```
15.235.203.23:80
```

the chain became:

```
cert-manager
    ↓
Gateway API HTTP-01 solver
    ↓
15.235.203.23:80
    ↓
iptables DNAT/SNAT
    ↓
Envoy NodePort :30237
    ↓
temporary ACME HTTPRoute
    ↓
HTTP-01 challenge
    ↓
Let's Encrypt
    ↓
Certificate issued
```

Then:

```
kubectl get challenges -n obs
```

returned:

```
No resources found
```

and the certificate became:

```
vapd-online-cert   True
```

So the **critical fix for the port-80 problem was the combination of external port-80 forwarding plus pod-CIDR DNAT + MASQUERADE for the internal hairpin path**.

---

### HTTPS curl — what we did

After HTTP/ACME was working, HTTPS initially failed:

```
curl https://grafana.vapd.online
→ 15.235.203.23:443
→ Connection refused
```

The reason was simple: **Envoy had HTTPS on a NodePort, but public port 443 was not forwarded to it.**

We checked:

```
kubectl get svc -n envoy envoy-obs-eg-gateway-02f355a3
```

and found:

```
80:30237/TCP
443:30732/TCP
```

So the traffic paths were:

```
HTTP:
Internet :80
    ↓
iptables
    ↓
NodePort :30237
    ↓
Envoy
```

For HTTPS we needed:

```
HTTPS:
Internet :443
    ↓
iptables
    ↓
NodePort :30732
    ↓
Envoy
    ↓
TLS termination
    ↓
Certificate: vapd-online-cert-secret
```

We therefore added:

```
sudo iptables -t nat -A PREROUTING \
  -i ens3 \
  -p tcp \
  -d 15.235.203.23 \
  --dport 443 \
  -j REDIRECT --to-port 30732
```

Then test:

```
curl -vk https://grafana.vapd.online
```

### Overall final architecture

```
                    Internet
                       |
              +--------+--------+
              |                 |
            :80               :443
              |                 |
           iptables          iptables
              |                 |
           :30237            :30732
              |                 |
              +--------+--------+
                       |
                  Envoy Gateway
                       |
              TLS termination :443
                       |
             vapd-online-cert-secret
                       |
                  HTTPRoute
                       |
                    Grafana
```

So the key distinction is:

**HTTP:** `80 → 30237`

**HTTPS:** `443 → 30732`

And TLS is terminated at **Envoy**, not at Grafana.

---

## Later
1. Prepare helm chart for app
2. Enable https access on gateway
