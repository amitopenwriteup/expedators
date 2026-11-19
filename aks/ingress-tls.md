# Lab: AKS Ingress with Host/Path Routing and TLS

**Goal:** Two apps behind one public IP, routed by hostname and by path, secured with HTTPS using cert-manager.

**Prereqs:** Running AKS cluster, `az`, `kubectl`, `helm`. No domain needed: `anything.<IP>.nip.io` resolves to `<IP>`.

> Note: community `ingress-nginx` is retired (no fixes after March 2026). It is used here only because the Ingress and TLS concepts carry over to Gateway API.

---

## Part 1: Connect and install the ingress controller

```bash
RG=<your-rg>
AKS=<your-cluster>

az aks get-credentials -g $RG -n $AKS --overwrite-existing
kubectl get nodes
```

Expected:
```
NAME                         STATUS   ROLES    AGE   VERSION
aks-nppool-12345678-vmss...  Ready    <none>   10d   v1.36.5
```

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

helm install ingress-nginx ingress-nginx/ingress-nginx \
  -n ingress-nginx --create-namespace \
  --set controller.replicaCount=2 \
  --set controller.service.annotations."service\.beta\.kubernetes\.io/azure-load-balancer-health-probe-request-path"=/healthz
```

Expected: `STATUS: deployed`

```bash
kubectl get svc ingress-nginx-controller -n ingress-nginx -w    # Ctrl+C when IP appears
```

Expected:
```
NAME                       TYPE           CLUSTER-IP     EXTERNAL-IP    PORT(S)
ingress-nginx-controller   LoadBalancer   10.0.123.45    20.x.x.x       80:3xxxx/TCP,443:3xxxx/TCP
```

```bash
export INGRESS_IP=$(kubectl get svc ingress-nginx-controller -n ingress-nginx \
  -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
echo $INGRESS_IP
kubectl get ingressclass
```

Expected:
```
20.x.x.x
NAME    CONTROLLER                      PARAMETERS   AGE
nginx   k8s.io/ingress-nginx            <none>       1m
```

---

## Part 2: Deploy two apps

```bash
kubectl create namespace demo
```

`apps.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata: { name: app1, namespace: demo }
spec:
  replicas: 2
  selector: { matchLabels: { app: app1 } }
  template:
    metadata: { labels: { app: app1 } }
    spec:
      containers:
      - name: app
        image: mcr.microsoft.com/azuredocs/aks-helloworld:v1
        ports: [{ containerPort: 80 }]
        env: [{ name: TITLE, value: "APP ONE" }]
        resources:
          requests: { cpu: 50m, memory: 64Mi }
          limits: { cpu: 200m, memory: 128Mi }
        readinessProbe:
          httpGet: { path: /, port: 80 }
          initialDelaySeconds: 3
---
apiVersion: v1
kind: Service
metadata: { name: app1, namespace: demo }
spec:
  selector: { app: app1 }
  ports: [{ port: 80, targetPort: 80 }]
---
apiVersion: apps/v1
kind: Deployment
metadata: { name: app2, namespace: demo }
spec:
  replicas: 2
  selector: { matchLabels: { app: app2 } }
  template:
    metadata: { labels: { app: app2 } }
    spec:
      containers:
      - name: app
        image: mcr.microsoft.com/azuredocs/aks-helloworld:v1
        ports: [{ containerPort: 80 }]
        env: [{ name: TITLE, value: "APP TWO" }]
        resources:
          requests: { cpu: 50m, memory: 64Mi }
          limits: { cpu: 200m, memory: 128Mi }
        readinessProbe:
          httpGet: { path: /, port: 80 }
          initialDelaySeconds: 3
---
apiVersion: v1
kind: Service
metadata: { name: app2, namespace: demo }
spec:
  selector: { app: app2 }
  ports: [{ port: 80, targetPort: 80 }]
```

```bash
kubectl apply -f apps.yaml
kubectl get pods -n demo
kubectl get endpoints -n demo
```

Expected:
```
NAME                    READY   STATUS    RESTARTS   AGE
app1-xxxxxxxxxx-aaaaa   1/1     Running   0          30s
app1-xxxxxxxxxx-bbbbb   1/1     Running   0          30s
app2-xxxxxxxxxx-ccccc   1/1     Running   0          30s
app2-xxxxxxxxxx-ddddd   1/1     Running   0          30s

NAME   ENDPOINTS                     AGE
app1   10.244.x.x:80,10.244.x.x:80   30s
app2   10.244.x.x:80,10.244.x.x:80   30s
```

---

## Part 3: Host-based routing

`ingress-host.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: host-routing
  namespace: demo
spec:
  ingressClassName: nginx
  rules:
  - host: app1.${INGRESS_IP}.nip.io
    http:
      paths:
      - path: /
        pathType: Prefix
        backend: { service: { name: app1, port: { number: 80 } } }
  - host: app2.${INGRESS_IP}.nip.io
    http:
      paths:
      - path: /
        pathType: Prefix
        backend: { service: { name: app2, port: { number: 80 } } }
```

```bash
envsubst '${INGRESS_IP}' < ingress-host.yaml | kubectl apply -f -
kubectl get ingress -n demo

curl -s http://app1.${INGRESS_IP}.nip.io | grep -i "APP ONE"
curl -s http://app2.${INGRESS_IP}.nip.io | grep -i "APP TWO"
curl -s -o /dev/null -w "%{http_code}\n" http://${INGRESS_IP}
```

Expected:
```
NAME           CLASS   HOSTS                                              ADDRESS      PORTS
host-routing   nginx   app1.20.x.x.x.nip.io,app2.20.x.x.x.nip.io          20.x.x.x     80

<title>APP ONE</title>
<title>APP TWO</title>
404
```

The 404 proves routing is by Host header: no hostname matches a bare IP.

---

## Part 4: Path-based routing

```bash
kubectl delete ingress host-routing -n demo
```

`ingress-path.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: path-routing
  namespace: demo
  annotations:
    nginx.ingress.kubernetes.io/use-regex: "true"
    nginx.ingress.kubernetes.io/rewrite-target: /$2
spec:
  ingressClassName: nginx
  rules:
  - host: shop.${INGRESS_IP}.nip.io
    http:
      paths:
      - path: /app1(/|$)(.*)
        pathType: ImplementationSpecific
        backend: { service: { name: app1, port: { number: 80 } } }
      - path: /app2(/|$)(.*)
        pathType: ImplementationSpecific
        backend: { service: { name: app2, port: { number: 80 } } }
```

```bash
envsubst '${INGRESS_IP}' < ingress-path.yaml | kubectl apply -f -

curl -s http://shop.${INGRESS_IP}.nip.io/app1/ | grep -i "APP ONE"
curl -s http://shop.${INGRESS_IP}.nip.io/app2/ | grep -i "APP TWO"
```

Expected:
```
<title>APP ONE</title>
<title>APP TWO</title>
```

The rewrite strips `/app1` before the request reaches the pod. Without it the pod returns 404.

---

## Part 5: Install cert-manager and create the issuer

```bash
helm repo add jetstack https://charts.jetstack.io
helm repo update

helm install cert-manager jetstack/cert-manager \
  -n cert-manager --create-namespace \
  --set crds.enabled=true

kubectl get pods -n cert-manager
```

Expected:
```
NAME                                       READY   STATUS    RESTARTS   AGE
cert-manager-xxxxxxxxxx-aaaaa              1/1     Running   0          60s
cert-manager-cainjector-xxxxxxxxxx-bbbbb   1/1     Running   0          60s
cert-manager-webhook-xxxxxxxxxx-ccccc      1/1     Running   0          60s
```

`issuer-staging.yaml` (change the email):

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-staging
spec:
  acme:
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    email: you@example.com
    privateKeySecretRef:
      name: letsencrypt-staging-key
    solvers:
    - http01:
        ingress:
          ingressClassName: nginx
```

```bash
kubectl apply -f issuer-staging.yaml
kubectl get clusterissuer
```

Expected:
```
NAME                  READY   AGE
letsencrypt-staging   True    15s
```

---

## Part 6: Enable TLS

`ingress-tls.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: path-routing
  namespace: demo
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-staging
    nginx.ingress.kubernetes.io/use-regex: "true"
    nginx.ingress.kubernetes.io/rewrite-target: /$2
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - shop.${INGRESS_IP}.nip.io
    secretName: shop-tls
  rules:
  - host: shop.${INGRESS_IP}.nip.io
    http:
      paths:
      - path: /app1(/|$)(.*)
        pathType: ImplementationSpecific
        backend: { service: { name: app1, port: { number: 80 } } }
      - path: /app2(/|$)(.*)
        pathType: ImplementationSpecific
        backend: { service: { name: app2, port: { number: 80 } } }
```

```bash
envsubst '${INGRESS_IP}' < ingress-tls.yaml | kubectl apply -f -
kubectl get certificate -n demo -w       # Ctrl+C when READY=True
```

Expected (after 30-90 seconds):
```
NAME       READY   SECRET     AGE
shop-tls   False   shop-tls   5s
shop-tls   True    shop-tls   45s
```

Inspect the issuance chain:

```bash
kubectl get certificaterequest,order,challenge -n demo
```

Expected: the CertificateRequest shows `APPROVED True` and the Order shows `valid`. The Challenge disappears once validation succeeds.

Test HTTPS:

```bash
curl -k https://shop.${INGRESS_IP}.nip.io/app1/ | grep -i "APP ONE"
curl -kv https://shop.${INGRESS_IP}.nip.io/app2/ 2>&1 | grep -iE "issuer|subject"
curl -sI http://shop.${INGRESS_IP}.nip.io/app1/ | head -2
```

Expected:
```
<title>APP ONE</title>
*  subject: CN=shop.20.x.x.x.nip.io
*  issuer: C=US; O=(STAGING) Let's Encrypt; CN=(STAGING) ...
HTTP/1.1 308 Permanent Redirect
Location: https://shop.20.x.x.x.nip.io/app1/
```

`-k` is needed because staging certificates are not trusted by clients. The `(STAGING)` issuer and the 308 redirect confirm TLS is working.

---

## Part 7: Cleanup

```bash
kubectl delete namespace demo
helm uninstall cert-manager -n cert-manager
kubectl delete clusterissuer letsencrypt-staging
helm uninstall ingress-nginx -n ingress-nginx
```

Expected:
```
namespace "demo" deleted
release "cert-manager" uninstalled
clusterissuer.cert-manager.io "letsencrypt-staging" deleted
release "ingress-nginx" uninstalled
```

Remove the leftover cert-manager CRDs and namespaces (Helm does not delete them):

```bash
kubectl delete namespace cert-manager ingress-nginx
kubectl get crd | grep cert-manager.io | awk '{print $1}' | xargs kubectl delete crd
```

Verify nothing is left:

```bash
kubectl get ns
kubectl get svc -A | grep LoadBalancer
```

Expected: no `demo`, `cert-manager` or `ingress-nginx` namespaces, and no LoadBalancer service from this lab. The public IP is released automatically when the ingress service is deleted.

Do **not** delete the resource group, since it holds your existing cluster.
