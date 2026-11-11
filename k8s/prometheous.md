# Lab: Install Prometheus and Grafana on Kubernetes with Helm (NFS Storage)

## Overview

In this lab you will install **Prometheus** (metrics collection) and **Grafana** (dashboards) on a Kubernetes cluster using Helm. Prometheus stores its data on NFS-backed PersistentVolumes created by the `nfs-client` StorageClass. You will expose both tools with NodePort Services and connect Grafana to Prometheus as a data source.

**Architecture**

```
 Browser
    |
    |  http://<NODE-IP>:<grafana-nodeport>
    v
+-----------+   data source URL    +--------------------+
|  Grafana  | -------------------> |  prometheus-server |
+-----------+                      +--------------------+
                                      |            |
                                 PVC (nfs-client)  scrapes metrics
                                      |            |
                                 NFS server    node-exporter,
                                               kube-state-metrics
```

**Objectives**

1. Search for Helm charts and add the Prometheus repository
2. Install Prometheus with Helm
3. Bind Prometheus PVCs to the `nfs-client` StorageClass
4. Expose Prometheus (port-forward and NodePort)
5. Install Grafana and retrieve the admin password
6. Expose Grafana with a NodePort and log in
7. Add Prometheus as a Grafana data source
8. Clean up

**Estimated time:** 30-45 minutes

## Prerequisites

| Requirement | Details |
|---|---|
| Kubernetes | Working cluster (1 control-plane VM and 2 worker VMs), `kubectl` configured |
| Helm | Helm 3 installed on the control-plane VM (see the NGINX Ingress lab, Part 1) |
| Storage | NFS server and the `nfs-client` StorageClass from the **NFS Storage Provisioner lab** |
| Network | Internet access to download charts and images |
| Firewall | If `firewalld` is running on the nodes, NodePorts (30000-32767) must be reachable |

Check the StorageClass before you start:

```bash
kubectl get storageclass
```

You should see `nfs-client` in the list.

> **Note:** The screenshots in this lab were taken on a cluster whose node IP is `192.168.111.130`. Your IP and NodePort numbers will differ. This lab uses `<NODE-IP>` as a placeholder; get your node IPs with `kubectl get nodes -o wide`.

---

## Part 1: Find and Add the Prometheus Helm Chart

### Step 1.1 - Search Artifact Hub from the command line

```bash
helm search hub prometheus
```

This lists many Prometheus-related charts from Artifact Hub:

![helm search hub output listing Prometheus charts](images/01-helm-search-hub.png)

### Step 1.2 - Check online

You can also search in the browser at <https://artifacthub.io/>. Search for `prometheus` and choose the chart from the **prometheus-community** repository (Helm chart, verified publisher).

![Artifact Hub search showing the prometheus chart from prometheus-community](images/02-artifacthub-prometheus.png)

### Step 1.3 - Add the repository and update

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```

Expected output ends with `Update Complete. Happy Helming!`

![helm repo update output showing prometheus-community updated](images/03-helm-repo-update.png)

---

## Part 2: Install Prometheus

### Step 2.1 - Install the chart

```bash
helm install prometheus prometheus-community/prometheus
```

### Step 2.2 - Check the Pods

```bash
kubectl get po
```

Wait until the Pods are `Running`. At first, the **server** and **alertmanager** Pods may stay `Pending` because their PVCs have no storage class yet. The next step fixes that.

### Step 2.3 - Check the PVCs

```bash
kubectl get pvc
```

You should see two PVCs:

| PVC | Used by |
|---|---|
| `prometheus-server` | Prometheus server data |
| `storage-prometheus-alertmanager-0` | Alertmanager data |

---

## Part 3: Bind the PVCs to the NFS StorageClass

### Step 3.1 - Edit the first PVC

```bash
kubectl edit pvc prometheus-server
```

Under `spec:`, add the storage class:

```yaml
spec:
  storageClassName: nfs-client
```

Save and exit.

### Step 3.2 - Edit the second PVC

```bash
kubectl edit pvc storage-prometheus-alertmanager-0
```

Add the same line under `spec:`:

```yaml
spec:
  storageClassName: nfs-client
```

### Step 3.3 - Verify

```bash
kubectl get pvc
kubectl get po
```

Both PVCs should be `Bound` and all Prometheus Pods `Running`.

> **Tip:** If `nfs-client` is already the **default** StorageClass in your cluster (it shows `(default)` in `kubectl get storageclass`), the PVCs bind automatically and no edit is needed.
>
> **Alternative:** Set the storage class at install time instead of editing PVCs afterwards:
>
> ```bash
> helm install prometheus prometheus-community/prometheus \
>   --set server.persistentVolume.storageClass=nfs-client \
>   --set alertmanager.persistence.storageClass=nfs-client
> ```
>
> Value names can change between chart versions; check them with `helm show values prometheus-community/prometheus`.

---

## Part 4: Access the Prometheus UI

Pick **one** of the two options.

### Option A: Port-forward

```bash
kubectl port-forward --address 0.0.0.0 svc/prometheus-server 9090:80
```

- `--address 0.0.0.0` makes the forward reachable from other machines, not only from localhost.
- Open `http://<CONTROL-PLANE-IP>:9090` in your browser.
- Keep the terminal open; press `Ctrl+C` to stop.
- If `firewalld` is running, allow the port: `sudo firewall-cmd --permanent --add-port=9090/tcp && sudo firewall-cmd --reload`

### Option B: Convert the Service to NodePort (used by Grafana later)

List the services:

```bash
kubectl get service
```

Edit the Prometheus server Service:

```bash
kubectl edit svc prometheus-server
```

Change `type: ClusterIP` to `type: NodePort`, then save.

> The original notes used `kubectl edit svc prometheus`. With the Helm release name `prometheus`, the Service created by this chart is named **`prometheus-server`**. Use the name shown in `kubectl get service`.

Find the assigned port:

```bash
kubectl get svc prometheus-server
```

Example: `80:30953/TCP`. Prometheus is now available at `http://<NODE-IP>:30953`.

---

## Part 5: Install Grafana

### Step 5.1 - Search and add the repository

```bash
helm search hub grafana
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
```

> If `helm repo add` reports that the repository is moved or deprecated, check the chart's page on <https://artifacthub.io/> for its current repository URL.

### Step 5.2 - Install the chart

```bash
helm install grafana grafana/grafana
```

### Step 5.3 - Get the admin password

```bash
kubectl get secret --namespace default grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo
```

Copy the output. The username is **`admin`**.

### Step 5.4 - Change the Service type to NodePort

```bash
kubectl edit svc grafana
```

Change `type: ClusterIP` to `type: NodePort`:

![vi editor showing the grafana Service with type NodePort](images/04-grafana-svc-nodeport.png)

Save and exit.

### Step 5.5 - Note the port

```bash
kubectl get svc
```

In this example, Grafana is exposed on NodePort **32208** (shown as `80:32208/TCP`):

![kubectl get svc showing grafana NodePort 32208](images/05-kubectl-get-svc-grafana.png)

---

## Part 6: Log in to Grafana

Open a browser and go to:

```
http://<NODE-IP>:<grafana-nodeport>
```

For example, `http://192.168.111.130:32208`. Log in with username `admin` and the password from Step 5.3.

After login, the Grafana **Home** page appears:

![Grafana home page welcome screen](images/06-grafana-home.png)

---

## Part 7: Add Prometheus as a Data Source

### Step 7.1 - Open Add data source

On the Home page, in the **Basic** panel, click **Add your first data source** (**DATA SOURCES**).

![Grafana Basic panel with Add your first data source card](images/07-grafana-add-datasource.png)

You can also go to **Connections > Data sources > Add data source**.

### Step 7.2 - Select Prometheus

Under **Time series databases**, click **Prometheus**.

![Add data source page with Prometheus listed under time series databases](images/08-grafana-select-prometheus.png)

### Step 7.3 - Provide the Prometheus URL

In **Connection**, enter the **Prometheus server URL**:

![Prometheus data source Connection section with server URL filled in](images/09-grafana-prometheus-url.png)

| Method | URL to enter |
|---|---|
| Using the NodePort (as in the screenshot) | `http://<NODE-IP>:<prometheus-nodeport>/`, for example `http://192.168.111.130:30953/` |
| Using in-cluster DNS (recommended; no NodePort needed) | `http://prometheus-server.default.svc.cluster.local:80` |

### Step 7.4 - Save and test

Scroll to the bottom of the page and click **Save & test**. A green message such as "Successfully queried the Prometheus API" confirms the connection.

### Step 7.5 - Try a query (optional)

Go to **Explore**, select the Prometheus data source, and run:

```
up
```

Every target that Prometheus scrapes should show a value of `1`.

For ready-made dashboards, go to **Dashboards > New > Import** and use a community dashboard ID such as **1860** (Node Exporter Full), then select your Prometheus data source.

---

## Part 8: Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Prometheus Pods `Pending` | PVCs have no storage class or NFS is unreachable | Complete Part 3; check `kubectl describe pvc <name>` and the NFS provisioner Pod |
| PVC edit rejected | The PVC already has a different storage class | Delete the release and PVCs, then reinstall using the `--set ...storageClass=nfs-client` alternative |
| Pods stuck in `ContainerCreating` | `nfs-utils` missing on a worker node | `sudo dnf install -y nfs-utils` on every node |
| Cannot open Grafana from the browser | NodePort blocked by firewall | `sudo firewall-cmd --permanent --add-port=<nodeport>/tcp && sudo firewall-cmd --reload` |
| Grafana login fails | Wrong password | Run the secret command in Step 5.3 again; username is `admin` |
| Grafana data source shows a connection error | Wrong URL or port | Verify with `kubectl get svc prometheus-server`; try the in-cluster DNS URL |
| Port-forward not reachable from another machine | Started without `--address 0.0.0.0`, or firewall | Restart with the flag; open the port in `firewalld` |

Useful commands:

```bash
kubectl describe pvc prometheus-server
kubectl logs deploy/prometheus-server -c prometheus-server
kubectl get pods -o wide
helm list
```

---

## Part 9: Cleanup

Uninstall Grafana and Prometheus using the **release name** only:

```bash
helm uninstall grafana
helm uninstall prometheus
```

> The original notes used `helm uninstall prometheus prometheus-community/prometheus`. `helm uninstall` takes only the release name.

PVCs created by StatefulSets (such as Alertmanager) may remain after uninstall. Remove them if you no longer need the data:

```bash
kubectl get pvc
kubectl delete pvc prometheus-server storage-prometheus-alertmanager-0
```

---

## Review Questions

1. What is the role of Prometheus, and what is the role of Grafana?
2. Why do the Prometheus server and Alertmanager need PersistentVolumeClaims?
3. What does the `nfs-client` StorageClass do when a PVC references it?
4. What is the difference between a ClusterIP and a NodePort Service?
5. Why is the in-cluster DNS name a good choice for the Grafana data source URL?
6. Why does `helm uninstall` use only the release name?

## Reference

- Grafana documentation: [Prometheus Operator configuration](https://grafana.com/docs/grafana-cloud/monitor-infrastructure/kubernetes-monitoring/configuration/config-other-methods/prometheus/prometheus-operator/)
- Artifact Hub: <https://artifacthub.io/>

## Summary

You added the Prometheus and Grafana Helm repositories, installed Prometheus with NFS-backed storage, exposed it through a port-forward or NodePort, installed Grafana, retrieved the admin password, exposed Grafana through a NodePort, and connected Grafana to Prometheus as a data source.
