# 🛠️ Troubleshooting CloudWatch Exporter, Grafana PVC, and EBS CSI Driver in AWS EKS Monitoring Stack

This document outlines the step-by-step debugging journey of setting up a monitoring stack on EKS using Prometheus and Grafana. It covers critical issues encountered with the **CloudWatch Exporter**, **Grafana Persistent Volume Claim (PVC)**, and **EBS CSI driver setup**.

---

## 🔧 Issue 1: CloudWatch Exporter Pod Crashing

### ❌ Error:

```text
Exception in thread "main" java.io.FileNotFoundException: --config.file=/config/config.yml (No such file or directory)
```

![CloudWatch Exporter Config Error](/ERR-IMAGES/debugging-prometheus-stack-cloudwatch-exporter-ebs-issues.jpg)

### ✅ Fix:

* Ensure the `configMap` is correctly mounted.
* Confirm the container startup path points to `/config/config.yml`.
* Use this structure in your `Deployment`:

```yaml
volumes:
  - name: cloudwatch-config
    configMap:
      name: cloudwatch-exporter-config
```

---

## 🔧 Issue 2: DNS Resolution Failure in Pods

### ❌ Error:

```text
lookup E045C991EA3F0C3D77E45495F073B964.sk1.us-east-1.eks.amazonaws.com: i/o timeout
```

![DNS Resolution Error](/ERR-IMAGES/debugging-prometheus-stack-cloudwatch-exporter-ebs-issues%202.jpg)

### ✅ Fix:

* Confirm node-level DNS is functional:

```bash
kubectl run dns-test --rm -i --tty --image=busybox --restart=Never -- sh
> nslookup amazon.com
```

![Busybox DNS Test](/ERR-IMAGES/debugging-prometheus-stack-cloudwatch-exporter-ebs-issues%203.jpg)

* Validate that the VPC has proper DNS hostnames/resolution enabled.
* Ensure the cluster’s CoreDNS pods are running:

```bash
kubectl get pods -n kube-system -l k8s-app=kube-dns
```

---

## 🔧 Issue 3: IAM Role Not Attached to Service Account

### ❌ Error:

```text
error looking up service account monitoring/cloudwatch-exporter-sa: serviceaccount "cloudwatch-exporter-sa" not found
```

![IAM Service Account Missing](/ERR-IMAGES/debugging-prometheus-stack-cloudwatch-exporter-ebs-issues%204.jpg)

### ✅ Fix:

* Recreate IAM service account with correct role:

```bash
eksctl create iamserviceaccount \
  --name cloudwatch-exporter-sa \
  --namespace monitoring \
  --cluster monitoring-cluster \
  --attach-policy-arn arn:aws:iam::<account-id>:policy/CloudWatchExporterPolicy \
  --approve --region us-east-1
```

* Ensure OIDC provider is associated:

```bash
eksctl utils associate-iam-oidc-provider --region=us-east-1 --cluster=monitoring-cluster --approve
```

![OIDC Setup Error](/ERR-IMAGES/debugging-prometheus-stack-cloudwatch-exporter-ebs-issues%205.jpg)

---

## 🔧 Issue 4: Grafana PVC Pending Due to Missing StorageClass

### ❌ Error:

```text
no persistent volumes available for this claim and no storage class is set
```

![PVC Pending Error](/ERR-IMAGES/debugging-prometheus-stack-cloudwatch-exporter-ebs-issues%206.jpg)

### ✅ Fix:

* Create a default StorageClass for EBS:

```bash
kubectl apply -f manifests/storage/gp2-default-storageclass.yaml
```

![Creating StorageClass](/ERR-IMAGES/debugging-prometheus-stack-cloudwatch-exporter-ebs-issues%207.jpg)

* Delete and recreate the PVC:

```bash
kubectl delete pvc -n monitoring prometheus-grafana
```

* Upgrade the Helm release:

```bash
helm upgrade prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --values manifests/prometheus/values.yaml
```

![PVC Bound](/ERR-IMAGES/debugging-prometheus-stack-cloudwatch-exporter-ebs-issues%208.jpg)

---

## 🔧 Issue 5: Missing EBS CSI Driver

### ❌ Error:

```text
StorageClass "gp2" is invalid: reclaimPolicy: Forbidden
```

![StorageClass Invalid](/ERR-IMAGES/debugging-prometheus-stack-cloudwatch-exporter-ebs-issues%209.jpg)

### ✅ Fix:

* Create IAM policy and attach it:

```bash
aws iam create-policy --policy-name AmazonEKS_EBS_CSI_Driver_Policy \
  --policy-document file://iam/ebs-csi-policy.json
```

![IAM Policy Creation](/ERR-IMAGES/debugging-prometheus-stack-cloudwatch-exporter-ebs-issues%2010.jpg)

* Create the service account:

```bash
eksctl create iamserviceaccount \
  --name ebs-csi-controller-sa \
  --namespace kube-system \
  --cluster monitoring-cluster \
  --attach-policy-arn arn:aws:iam::<account-id>:policy/AmazonEKS_EBS_CSI_Driver_Policy \
  --approve --region us-east-1
```

* Install the driver:

```bash
helm repo add aws-ebs-csi-driver https://kubernetes-sigs.github.io/aws-ebs-csi-driver
helm upgrade --install aws-ebs-csi-driver aws-ebs-csi-driver/aws-ebs-csi-driver \
  --namespace kube-system \
  --set controller.serviceAccount.name=ebs-csi-controller-sa \
  --set controller.serviceAccount.create=false
```

---

## ✅ Final Result

* CloudWatch Exporter is running
* Grafana PVC is `Bound`
* EBS CSI Driver is installed and functional

> You can now continue to build dashboards and configure alerts with confidence. 🧠📈
