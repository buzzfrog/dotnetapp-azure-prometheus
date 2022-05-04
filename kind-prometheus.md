# Kind
This document outline a test to use Kind as the local cluster, and in combination
with Prometheus and Azure Arc for Kubernetes, get metrics to Azure Monitor.

## Prerequisites
* [Kind](https://kind.sigs.k8s.io/)
* az (with arc extensions)
```
az extension add --name connectedk8s
```

## Installation
1. Create a new Kind cluster
```
kind create cluster --name contoso
```

2. Install Prometheus
```

helm repo add stable https://charts.helm.sh/stable

helm repo add prometheus-community https://prometheus-community.github.io/helm-charts

helm repo add kube-state-metrics https://kubernetes.github.io/kube-state-metrics

helm repo update

helm install prometheus-contoso prometheus-community/prometheus
```

3. Create Resource Group
```
 az group create --name contoso-rg --location <location-of-choice>
```

4. Add cluster to Azure Arc for Kubernetes
```
az connectedk8s connect --name contoso --resource-group contoso-rg
```

5. Create a Log Analytics Workspace to store our metrics
```
az monitor log-analytics workspace create -g contoso-rg -n contoso-log -o json --query "id"
```

6. Enable Azure Monitor for containers extension
```
az k8s-extension create --name azuremonitor-containers --cluster-name contoso --resource-group contoso-rg --cluster-type connectedClusters --extension-type Microsoft.AzureMonitor.Containers --configuration-settings logAnalyticsWorkspaceResourceID=<id-from-above-command>
```
When this command is completed, wait 5-10 minutes before any metrics is displayed in Azure Portal.

 ![insights-metrics](./assets/azure-monitor.png)

7. Deploy our application (we use a predefined image in docker hub)
```
kubectl apply -f ./deploy/dep.yaml
```

8. Update Prometheus to get metrics from our application
```
kubectl apply -f Application/manifests/container-azm-ms-agentconfig.yaml
```

9. Do port forwarding to our application
```
# get pod name
kubectl get po -l app=milkyway-sample
kubectl port-forward <pod-name> 8080:8080
```
10. Open browser to http://localhost:8080 and change pages to create some metrics

11. Do a KQL query
```
InsightsMetrics
| where Name == "prom_counter_request_total"
| where parse_json(Tags).method == "GET"
| extend path = parse_json(Tags).path
```
![insights-metrics](./assets/insights-metrics2.png)