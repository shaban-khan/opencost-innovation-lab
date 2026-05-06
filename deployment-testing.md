Excellent — this is a real DevOps/FinOps production task, and AKS has some Azure-specific pieces that many tutorials skip.

I’ll give you:

* **Architecture understanding first**
* **Prerequisites**
* **Full OpenCost installation in AKS**
* **Prometheus setup**
* **Azure pricing integration (VERY IMPORTANT)**
* **Verification**
* **UI access**
* **Troubleshooting**
* **Production best practices**

This is based on official OpenCost Helm deployment guidance + Azure AKS integration recommendations. ([OpenCost][1])

---

# 1. First Understand: What OpenCost Needs to Work in AKS

OpenCost does **not magically know AKS costs**.

It needs **3 major data sources**:

| Component                | Why Needed                      |
| ------------------------ | ------------------------------- |
| Kubernetes Metrics       | CPU, Memory, PVC, Network usage |
| Prometheus               | Stores those metrics            |
| Azure Cloud Pricing Info | To convert usage → ₹/$ cost     |

So architecture becomes:

```text
AKS Cluster
   |
   |-- kube-state-metrics
   |-- node-exporter/cAdvisor metrics
   |-- Prometheus
   |-- OpenCost
   |
   ---> OpenCost reads Prometheus metrics
   ---> OpenCost reads Azure pricing/rate card
   ---> Calculates pod/namespace/deployment costs
```

Without Prometheus + Azure pricing = OpenCost reports are incomplete.

---

# 2. Prerequisites

You need these installed on your admin machine:

### Required Tools

```bash
az version
kubectl version --client
helm version
```

If kubectl not connected:

```bash
az login
az account set --subscription "<SUBSCRIPTION_ID>"
az aks get-credentials -g <RESOURCE_GROUP> -n <AKS_CLUSTER_NAME>
```

Verify cluster access:

```bash
kubectl get nodes
```

---

# 3. Create Namespace for Monitoring Stack

We usually keep monitoring tools isolated.

```bash
kubectl create namespace monitoring
kubectl create namespace opencost
```

---

# 4. Install Prometheus in AKS (Mandatory)

OpenCost official Helm chart assumes an existing Prometheus source. ([OpenCost][1])

---

## Add Prometheus Helm repo

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```

---

## Install kube-prometheus-stack

This installs:

* Prometheus
* Alertmanager
* kube-state-metrics
* node-exporter
* Grafana (optional but useful)

```bash
helm install monitoring prometheus-community/kube-prometheus-stack \
  --namespace monitoring
```
output:

```text
NAME: monitoring
LAST DEPLOYED: Tue May  5 04:11:59 2026
NAMESPACE: monitoring
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
kube-prometheus-stack has been installed. Check its status by running:
  kubectl --namespace monitoring get pods -l "release=monitoring"

Get Grafana 'admin' user password by running:

  kubectl --namespace monitoring get secrets monitoring-grafana -o jsonpath="{.data.admin-password}" | base64 -d ; echo

Access Grafana local instance:

  export POD_NAME=$(kubectl --namespace monitoring get pod -l "app.kubernetes.io/name=grafana,app.kubernetes.io/instance=monitoring" -oname)
  kubectl --namespace monitoring port-forward $POD_NAME 3000

Get your grafana admin user password by running:

  kubectl get secret --namespace monitoring -l app.kubernetes.io/component=admin-secret -o jsonpath="{.items[0].data.admin-password}" | base64 --decode ; echo


Visit https://github.com/prometheus-operator/kube-prometheus for instructions on how to create & configure Alertmanager and Prometheus instances using the Operator.
```

Wait for pods:

```bash
kubectl get pods -n monitoring
```

Expected pods:

```text
monitoring-kube-prometheus-prometheus-0
monitoring-kube-state-metrics
monitoring-prometheus-node-exporter-xxxx
monitoring-grafana-xxxx
```

---

# 5. Verify Prometheus Service Name (Needed by OpenCost)

Run:

```bash
kubectl get svc -n monitoring
```

You will see something like:

```text
monitoring-kube-prometheus-prometheus   ClusterIP   9090/TCP

NAME                                      TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)                      AGE
alertmanager-operated                     ClusterIP   None           <none>        9093/TCP,9094/TCP,9094/UDP   35m
monitoring-grafana                        ClusterIP   10.0.93.187    <none>        80/TCP                       35m
monitoring-kube-prometheus-alertmanager   ClusterIP   10.0.236.210   <none>        9093/TCP,8080/TCP            35m
monitoring-kube-prometheus-operator       ClusterIP   10.0.22.41     <none>        443/TCP                      35m
monitoring-kube-prometheus-prometheus     ClusterIP   10.0.15.25     <none>        9090/TCP,8080/TCP            35m
monitoring-kube-state-metrics             ClusterIP   10.0.126.149   <none>        8080/TCP                     35m
monitoring-prometheus-node-exporter       ClusterIP   10.0.35.199    <none>        9100/TCP                     35m
prometheus-operated                       ClusterIP   None           <none>        9090/TCP                     35m
```

Remember this service name.

---

# 6. Create Azure Service Principal for Pricing API Access

This is the Azure-specific MOST IMPORTANT part.

OpenCost needs Azure billing/rate information to calculate real AKS cost instead of dummy estimates. ([OpenCost][2])

Create SP:

```bash
az ad sp create-for-rbac --name opencost-sp --role Reader --scopes /subscriptions/<SUBSCRIPTION_ID>
```

Save:

* appId = client ID
* password = client secret
* tenant = tenant ID

Also get subscription ID:

```bash
az account show --query id -o tsv
```

---

# 7. Create Kubernetes Secret for Azure Credentials

## 7.1 Create file

```bash
vi cloud-integration.json
```

Paste:
```json
{
  "subscriptionId": "<SUBSCRIPTION_ID>",
  "tenantId": "<TENANT_ID>",
  "clientId": "<APP_ID>",
  "clientSecret": "<CLIENT_SECRET>"
}
```
## 7.2 Create secret in Kubernetes

```bash
kubectl create secret generic azure-service-key \
  --from-file=cloud-integration.json \
  -n opencost
```


Verify:

```bash
kubectl get secret -n opencost
```

---

# 8. Add OpenCost Helm Repository

```bash
helm repo add opencost-charts https://opencost.github.io/opencost-helm-chart
helm repo update
```

---

# 9. Create OpenCost values.yaml (AKS Custom Configuration)

Create file:

```bash
vi values-opencost.yaml
```

Put this:

```yaml
opencost:
  exporter:
    defaultClusterId: aks-prod-cluster

  prometheus:
    internal:
      enabled: true
      namespaceName: monitoring
      serviceName: monitoring-kube-prometheus-prometheus
      port: 9090

  ui:
    enabled: true

  cloudIntegrationSecret: azure-service-key

  cloudCost:
    enabled: true

  dataRetention:
    dailyResolutionDays: 30

  customPricing:
    enabled: true
    provider: azure

service:
  type: ClusterIP
```

---

## Explanation of each parameter

| Parameter              | Meaning                          |
| ---------------------- | -------------------------------- |
| defaultClusterId       | Your cluster name shown in UI    |
| namespaceName          | Prometheus namespace             |
| serviceName            | Prometheus service               |
| port                   | Prometheus port                  |
| cloudIntegrationSecret | Azure SP secret                  |
| cloudCost.enabled      | Pull Azure cloud pricing         |
| customPricing.provider | tells OpenCost provider is Azure |
| ui.enabled             | enable dashboard                 |

---

# 10. Install OpenCost in AKS

```bash
helm install opencost opencost-charts/opencost \
  --namespace opencost \
  -f values-opencost.yaml
```

output:

```text
NAME: opencost
LAST DEPLOYED: Tue May  5 05:01:41 2026
NAMESPACE: opencost
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
Thank you for installing the OpenCost Helm Chart!

Your release is named: opencost

To verify that OpenCost is running, execute the following commands:

1. Check the status of the pods:
   kubectl get pods -l app.kubernetes.io/instance=opencost -n opencost

2. Access the OpenCost UI:

   If the service is only accessible within the cluster, you can use port-forwarding:
   export SVC_NAME=opencost
   kubectl port-forward svc/$SVC_NAME 9090 -n opencost

   Alternatively, you can port-forward directly to the OpenCost pod:
   export POD_NAME=$(kubectl get pod -l app.kubernetes.io/instance=opencost -n opencost -o jsonpath="{.items[0].metadata.name}")
   kubectl -n opencost port-forward $POD_NAME 9090

For more information, visit the OpenCost documentation:
https://www.opencost.io/docs/
```

Official OpenCost deployment is Helm based. ([OpenCost][1])


---

# 11. Verify OpenCost Pods

```bash
kubectl get pods -n opencost
```

Expected:

```text
opencost-xxxxxxxxx
opencost-ui-xxxxxxxx
```

Also check services:

```bash
kubectl get svc -n opencost
```
ouotput:

```text
NAME       TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)                      AGE
opencost   ClusterIP   10.0.154.6   <none>        9003/TCP,8081/TCP,9090/TCP   70m

---

# 12. Check OpenCost Logs (Very Important)

```bash
kubectl logs -n opencost deploy/opencost
```

Look for:

```text
Using provider Azure
Connected to Prometheus
Cloud cost integration enabled
```

If Azure auth successful you should not see credential errors.

---

# 13. Access OpenCost Dashboard

Port forward:

```bash
kubectl port-forward -n opencost svc/opencost 9090:9090
```

Open browser:

```text
http://localhost:9090
```

You will get:

* Cluster Assets
* Namespace Cost
* Deployment Cost
* Pod Cost
* PV Cost
* Idle Cost

---

# 14. Verify OpenCost API (Useful for DevOps)

```bash
curl http://localhost:9003/allocation/compute?window=24h
```

or

```bash
curl http://localhost:9003/model/assets
```

This returns JSON cost allocation data.

Useful for:

* chargeback
* showback
* FinOps automation
* monthly reports

---

# 15. What OpenCost Actually Calculates in AKS

After 15–30 mins data collection, it shows:

| Cost Category            | Visible? |
| ------------------------ | -------- |
| Node VM Cost             | Yes      |
| CPU cost per namespace   | Yes      |
| Memory cost per pod      | Yes      |
| PVC disk cost            | Yes      |
| LoadBalancer shared cost | Yes      |
| Idle cluster waste       | Yes      |
| Namespace wise spend     | Yes      |
| Deployment wise spend    | Yes      |

---

# 16. Recommended Production Enhancement (Highly Recommended)

Instead of port-forward, expose with internal ingress:

```bash
kubectl expose svc opencost -n opencost --type=LoadBalancer --port=9003
```

Or create ingress with authentication.

---

# 17. Common Problems in AKS OpenCost Installation

---

## Problem 1: No cost data showing

Cause:

* Prometheus not reachable

Check:

```bash
kubectl exec -it -n opencost deploy/opencost -- wget -qO- http://monitoring-kube-prometheus-prometheus.monitoring.svc:9090
```

---

## Problem 2: Wrong fake prices shown

Cause:

* Azure SP not configured
* Azure RateCard inaccessible

OpenCost uses default prices if Azure pricing missing. This is common issue in AKS. ([GitHub][3])

---

## Problem 3: Empty namespace allocations

Wait 15–20 min because Prometheus needs scrape history.

---

# 18. AKS Native Alternative You Should Know

Azure now also provides:

```bash
az aks update --resource-group <rg> --name <cluster> --enable-cost-analysis
```

This enables Azure Portal AKS Cost Analysis built on OpenCost engine. ([Microsoft Learn][4])

BUT:

### Native AKS Cost Analysis limitations:

* less customizable
* no self-hosted API control
* no advanced FinOps automation

Self-hosted OpenCost = more flexible.

---

# 19. Real Production Recommendation From Me

For enterprise AKS, best practice is:

```text
Prometheus + OpenCost + Grafana + Azure Cost Export
```

This gives:

* live K8s usage cost
* actual Azure billing correlation
* team chargeback
* idle waste detection
* rightsizing

---

# 20. Complete Command Summary (Quick Runbook)

```bash
az aks get-credentials -g <rg> -n <cluster>

kubectl create ns monitoring
kubectl create ns opencost

helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm install monitoring prometheus-community/kube-prometheus-stack -n monitoring

az ad sp create-for-rbac --name opencost-sp --role Reader --scopes /subscriptions/<SUB_ID>

kubectl create secret generic azure-service-key \
  --from-literal=AZURE_SUBSCRIPTION_ID=<SUB_ID> \
  --from-literal=AZURE_TENANT_ID=<TENANT_ID> \
  --from-literal=AZURE_CLIENT_ID=<APP_ID> \
  --from-literal=AZURE_CLIENT_SECRET=<PASSWORD> \
  -n opencost

helm repo add opencost-charts https://opencost.github.io/opencost-helm-chart
helm repo update

helm install opencost opencost-charts/opencost -n opencost -f values-opencost.yaml

kubectl port-forward -n opencost svc/opencost 9003:9003
```

---

# 21. I can also provide you a FULL REAL-TIME LAB GUIDE

I can give you:

* exact production-ready YAML,
* exact values-opencost.yaml,
* screenshots expected,
* every command with sample outputs,
* and AKS architecture diagram

just like a hands-on implementation workbook.

Just say **continue** 👇

[1]: https://opencost.io/docs/installation/helm/?utm_source=chatgpt.com "Managing with Helm"
[2]: https://opencost.io/docs/configuration/azure/?utm_source=chatgpt.com "Installing on Microsoft Azure"
[3]: https://github.com/opencost/opencost/issues/2972?utm_source=chatgpt.com "Opencost on Azure/AKS fails to download Cost-Data from ..."
[4]: https://learn.microsoft.com/en-us/azure/aks/cost-analysis?utm_source=chatgpt.com "Azure Kubernetes Service (AKS) cost analysis"
