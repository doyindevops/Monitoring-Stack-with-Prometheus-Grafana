# Debugging Prometheus Monitoring Stack on AWS EKS

> All screenshots are saved in the root directory under `ERR-IMAGES/`

---

## Overview

This document walks through the troubleshooting journey of deploying a production-grade monitoring stack on AWS EKS using Prometheus, Grafana, and supporting components like CloudWatch Exporter and EBS CSI Driver.

We encountered multiple deployment issues across:

* Missing IAM/OIDC configuration
* CrashLoopBackOff from misconfigured exporters
* PVC binding issues
* EBS CSI driver not properly installed

---

## Issue 1: CloudWatch Exporter Pod in `CrashLoopBackOff`

### Error Summary

```
Exception in thread "main" java.io.FileNotFoundException: --config.file=/config/config.yml (No such file or directory)
```

### Root Cause

The CloudWatch exporter deployment was missing the required configuration file in the expected mount path.

### Fix

* Defined a ConfigMap named `cloudwatch-exporter-config`.
* Mounted the ConfigMap at `/config/config.yml` in the deployment YAML.

Screenshot: `ERR-IMAGES/debugging-prometheus-stack-cloudwatch-exporter-ebs-issues.jpg`

---

## Issue 2: DNS Resolution Failures

### Error Summary

```
dial tcp: lookup oidc.eks.us-east-1.amazonaws.com ... i/o timeout
```

### Root Cause

Kubernetes nodes had DNS misconfiguration or transient networking issues.

### Fix

Tested DNS from within a busybox pod:

```bash
kubectl run dns-test --rm -i --tty --image=busybox --restart=Never -- sh
/ # nslookup amazon.com
```

Confirmed DNS resolution worked from the pod.

Screenshot: `ERR-IMAGES/debugging-prometheus-stack-cloudwatch-exporter-ebs-issues 2.jpg`

---

## Issue 3: IAM OIDC Not Associated

### Error Summary

```
no IAM OIDC provider associated with cluster
```

### Fix

Associated IAM OIDC to EKS cluster:

```bash
eksctl utils associate-iam-oidc-provider \
  --region us-east-1 \
  --cluster monitoring-cluster \
  --approve
```

Screenshot: `ERR-IMAGES/debugging-prometheus-stack-cloudwatch-exporter-ebs-issues 3.jpg`

---

## Issue 4: ServiceAccount Missing for CloudWatch Exporter

### Error Summary

```
Error creating pod: serviceaccount "cloudwatch-exporter-sa" not found
```

### Fix

* Created IAM policy with permissions:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "cloudwatch:GetMetricData",
        "cloudwatch:GetMetricStatistics",
        "cloudwatch:ListMetrics"
      ],
      "Resource": "*"
    }
  ]
}
```

* Created IAM role with trust relationship to OIDC
* Re-created `cloudwatch-exporter-sa` using `eksctl`

Screenshot: `ERR-IMAGES/debugging-prometheus-stack-cloudwatch-exporter-ebs-issues 4.jpg`

---

## Issue 5: PVCs in Pending State (Grafana, Prometheus)

### Error Summary

```
no persistent volumes available for this claim and no storage class is set
```

### Fix

* Defined `gp2-default` storage class:

```yaml
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: gp2-default
provisioner: kubernetes.io/aws-ebs
volumeBindingMode: WaitForFirstConsumer
```

* Deleted existing stuck PVCs
* Re-deployed Prometheus stack via Helm upgrade

Screenshot: `ERR-IMAGES/debugging-prometheus-stack-cloudwatch-exporter-ebs-issues 5.jpg`

---

## Issue 6: EBS CSI Driver Missing / Unbound Volumes

### Error Summary

```
PVCs bound to non-existent volume provisioner. No volumes created.
```

### Fix

* Created IAM policy `AmazonEKS_EBS_CSI_Driver_Policy` using:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ec2:CreateVolume",
        "ec2:DeleteVolume",
        "ec2:AttachVolume",
        "ec2:DetachVolume",
        "ec2:ModifyVolume",
        "ec2:DescribeAvailabilityZones",
        "ec2:DescribeInstances",
        "ec2:DescribeVolumes",
        "ec2:DescribeVolumeAttribute",
        "ec2:DescribeVolumeStatus",
        "ec2:DescribeSnapshots",
        "ec2:CreateTags",
        "ec2:DeleteTags"
      ],
      "Resource": "*"
    }
  ]
}
```

* Created service account `ebs-csi-controller-sa` and attached IAM role
* Installed EBS CSI driver via Helm

Screenshot: `ERR-IMAGES/debugging-prometheus-stack-cloudwatch-exporter-ebs-issues 6.jpg`

---

## Final Resolution

After all fixes:

```bash
kubectl get pods -n monitoring
```

All pods now in `Running` state:

* `prometheus-grafana` (3/3)
* `prometheus-kube-state-metrics`
* `prometheus-node-exporter`
* `cloudwatch-exporter`
* `ebs-csi-node`, `ebs-csi-controller` (kube-system)

Final Screenshots: `ERR-IMAGES/debugging-prometheus-stack-cloudwatch-exporter-ebs-issues 9.jpg`, `10.jpg`

---

## Lessons Learned

| Problem Area         | What to Watch For               |
| -------------------- | ------------------------------- |
| IAM Service Accounts | Ensure OIDC is enabled first    |
| ConfigMaps           | Always validate mount paths     |
| PVCs                 | Set default storage class early |
| CloudWatch Exporter  | Requires explicit config file   |

---

## Summary

You now have:

* Full observability stack running on AWS EKS
* All exporters configured and metrics flowing
* Persistent volumes working with EBS CSI

Ready to visualize and alert
