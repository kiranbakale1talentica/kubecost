# Kubecost

Kubecost is a cost monitoring and optimization platform designed for Kubernetes environments. It provides real-time insights into cluster costs, resource utilization, and efficiency. By integrating seamlessly with Kubernetes, Kubecost helps allocate costs, reduce resource waste, and optimize resource usage across workloads, namespaces, or teams. It supports hybrid and multi-cloud environments, enabling effective cost management across diverse infrastructures.

---

## Key Features

- **Cost Allocation**: Detailed tracking of resource costs by namespace, service, deployment, or pod.
- **Multi-Cloud and Hybrid Support**: Visibility and cost management for Kubernetes workloads across various cloud providers and on-premises setups.
- **Real-Time Insights**: Live cost and resource metrics for quick decision-making.
- **Resource Efficiency Recommendations**: Actionable suggestions to optimize underutilized or over-provisioned resources.
- **Custom Cost Settings**: Support for unique discounts, licenses, or infrastructure costs.
- **Alerting and Notifications**: Alerts for cost anomalies, overspending, or efficiency thresholds.
- **API and Integrations**: Seamless integration with tools like Prometheus, Grafana, and cloud cost tools.
- **Historical Reporting**: Insights into trends and budget adherence over time.

---

## Prerequisites

- **Helm Client** (version 3.1+): [Install Here](https://helm.sh/docs/intro/install/)
- **Kubectl**: [Install Here](https://docs.aws.amazon.com/eks/latest/userguide/install-kubectl.html#kubectl-install-update)
- A supported Kubernetes cluster (EKS version 1.23+ requires the Amazon EBS CSI driver).
- Note: For EKS version 1.30 or later, a default StorageClass is no longer assigned.

---

## Why Kubecost?

| Feature                     | OpenCost                          | Kubecost Free                    | Kubecost Enterprise                   |
|-----------------------------|-----------------------------------|----------------------------------|---------------------------------------|
| **Description**             | Open-source cost monitoring       | Includes OpenCost features with better scaling and savings options | Full-featured with advanced integrations and support |
| **Best For**                | Small clusters                   | Medium clusters                 | Large-scale or complex infrastructure |
| **Clusters Supported**      | Unlimited (no unified view)      | Unlimited (no unified view)     | Unified multi-cluster view            |
| **Deployment**              | Pod-based                        | Helm-based                      | Helm-based                            |
| **Metric Retention**        | Limited by Prometheus environment | 15 days                        | Unlimited (supports object storage)  |
| **Support**                 | Community-driven                 | Ticket-based                    | Enterprise-level support              |

---

## How Kubecost Works

![Screenshot 2025-01-26 135236](https://github.com/user-attachments/assets/1492b6ae-d6c1-4a2a-81fc-859803f16dc3)

The Kubecost Helm chart deploys major components such as:
- **Kubecost Cost-Analyzer Pod**:
  - Frontend: Manages routing and UI.
  - Cost-Model: Performs cost allocation calculations.
- **Prometheus**:
  - Time-series data store for cost and health metrics.
  - Includes kube-state-metrics, node-exporter, and Alertmanager.
- **Grafana**:
  - Dashboards for cost and efficiency monitoring.

Kubecost retrieves AWS pricing data, integrates it with Prometheus metrics, and provides actionable cost insights via its dashboard.

![image](https://github.com/user-attachments/assets/98b7e90e-f18f-492e-a03b-b3c9369fc487)


---

## Steps to Perform POC on AWS EKS

1. **Create an EKS Cluster**:
   ```bash
   eksctl create cluster --name my-cluster --version 1.30 --region us-east-1 --nodegroup-name my-nodes --node-type t3.medium --nodes 1 --nodes-min 1 --nodes-max 1 --managed
   ```

2. **Update kubeconfig**:
   ```bash
   aws eks update-kubeconfig --region us-east-1 --name my-cluster
   ```

3. **Install AWS EBS CSI Driver**:
   a. Associate/Enable IAM-OIDC-Provider
   ```bash
      eksctl utils associate-iam-oidc-provider --region=us-east-1 --cluster=my-cluster --approve
   ```
   
   a. Create an IAM service account:
   ```bash
   eksctl create iamserviceaccount \
       --name ebs-csi-controller-sa \
       --namespace kube-system \
       --cluster my-cluster \
       --attach-policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy \
       --approve \
       --role-only \
       --role-name AmazonEKS_EBS_CSI_DriverRole
   ```
   b. Install the add-on:
   ```bash
   eksctl create addon --name aws-ebs-csi-driver --cluster my-cluster \
       --service-account-role-arn $(aws iam get-role --role-name AmazonEKS_EBS_CSI_DriverRole --output json | jq -r '.Role.Arn') --force
   ```

5. **Install Kubecost**:
   ```bash
   helm install kubecost cost-analyzer --repo https://kubecost.github.io/cost-analyzer/     --namespace kubecost --create-namespace --set prometheus.server.persistentVolume.storageClass="gp2" --set persistentVolume.storageClass="gp2""
   ```

6. **Access the Dashboard**:
   Enable port-forwarding to expose the dashboard:
   ```bash
   kubectl port-forward --namespace kubecost deployment/kubecost-cost-analyzer 9090
   ```

**We can also install kubecost using using plain manifest you can skip the helm steps above and apply manifest below**
1. Clone the repository
```bash
kubectl apply -f kubecost.yaml
```
2. Enable port-forwarding to expose the dashboard:
   ```bash
   kubectl port-forward --namespace kubecost deployment/kubecost-cost-analyzer 9090
   ```


please check the below repo for more information on helm-values

https://github.com/kubecost/cost-analyzer-helm-chart

https://github.com/kubecost/cost-analyzer-helm-chart?tab=readme-ov-file#common-parameters
## Common Parameters

The following table lists commonly used configuration parameters for the Kubecost Helm chart and their default values. Please see the values file for the complete set of definable values.

Parameter | Description | Default
--------- | ----------- | -------
`global.prometheus.enabled` | If false, use an existing Prometheus install. [More info](https://docs.kubecost.com/install-and-configure/install/custom-prom). | `true`
`prometheus.server.persistentVolume.enabled` | If true, Prometheus server will create a Persistent Volume Claim. | `true`
`prometheus.server.persistentVolume.size` | Prometheus server data Persistent Volume size. Default set to retain ~6000 samples per second for 15 days. | `32Gi`
`prometheus.server.persistentVolume.storageClass` | Define storage class for Prometheus persistent volume  | `-`
`prometheus.server.retention` | Determines when to remove old data. | `97h`
`prometheus.server.resources` | Prometheus server resource requests and limits. | `{}`
`prometheus.nodeExporter.resources` | Node exporter resource requests and limits. | `{}`
`prometheus.nodeExporter.enabled` `prometheus.serviceAccounts.nodeExporter.create` | If false, do not create NodeExporter daemonset.  | `true`
`prometheus.alertmanager.persistentVolume.enabled` | If true, Alertmanager will create a Persistent Volume Claim. | `false`
`prometheus.pushgateway.persistentVolume.enabled` | If true, Prometheus Pushgateway will create a Persistent Volume Claim. | `false`
`persistentVolume.enabled` | If true, Kubecost will create a Persistent Volume Claim for product config data.  | `true`
`persistentVolume.size` | Define PVC size for cost-analyzer  | `32.0Gi`
`persistentVolume.dbSize` | Define PVC size for cost-analyzer's flat file database  | `32.0Gi`
`persistentVolume.storageClass` | Define storage class for cost-analyzer's persistent volume  | `-`
`ingress.enabled` | If true, Ingress will be created | `false`
`ingress.annotations` | Ingress annotations | `{}`
`ingress.className` | Ingress class name | `{}`
`ingress.paths` | Ingress paths | `["/"]`
`ingress.hosts` | Ingress hostnames | `[cost-analyzer.local]`
`ingress.tls` | Ingress TLS configuration (YAML) | `[]`
`networkCosts.enabled` | If true, collect network allocation metrics [More info](https://docs.kubecost.com/using-kubecost/navigating-the-kubecost-ui/cost-allocation/network-allocation) | `false`
`networkCosts.podMonitor.enabled` | If true, a PodMonitor for the network-cost daemonset is created | `false`
`serviceMonitor.enabled` | Set this to `true` to create ServiceMonitor for Prometheus operator | `false`
`serviceMonitor.additionalLabels` | Additional labels that can be used so ServiceMonitor will be discovered by Prometheus | `{}`
`prometheusRule.enabled` | Set this to `true` to create PrometheusRule for Prometheus operator | `false`
`prometheusRule.additionalLabels` | Additional labels that can be used so PrometheusRule will be discovered by Prometheus | `{}`
`grafana.resources` | Grafana resource requests and limits. | `{}`
`grafana.sidecar.dashboards.enabled` | Set this to `false` to disable creation of Dashboards in Grafana | `true`
`grafana.sidecar.datasources.defaultDatasourceEnabled` | Set this to `false` to disable creation of Prometheus datasource in Grafana | `true`
`serviceAccount.create` | Set this to `false` if you want to create the service account `kubecost-cost-analyzer` on your own | `true`
`tolerations` | node taints to tolerate | `[]`
`affinity` | pod affinity | `{}`
`extraVolumes` | A list of volumes to be added to the pod | `[]`|
`extraVolumeMounts` | A list of volume mounts to be added to the pod | `[]`


---

## Clean-Up

- **Uninstall Kubecost**:
  ```bash
  helm uninstall kubecost -n kubecost
  ```

- **Delete the EKS Cluster**:
  ```bash
  eksctl delete cluster --name my-cluster --region us-east-1
  ```

---

## Additional Resources

- [Official Documentation](https://www.kubecost.com/install.html#show-instructions)
- [AWS Blog Post](https://aws.amazon.com/blogs/containers/aws-and-kubecost-collaborate-to-deliver-cost-monitoring-for-eks-customers/) 
- [Medium Article on Kubecost](https://blog.searce.com/deploy-kubecost-on-eks-8462e1ed960b)
