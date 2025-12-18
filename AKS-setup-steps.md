
# 🔧 AWS managed EKS Setup Guide on Linux 

This guide installs and configures the AWS managed EKS cluster, and sets up EKS with EBS CSI driver, NGINX Ingress, and cert-manager.


## 📦 Install AWS CLI

```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
sudo apt install unzip -y
unzip awscliv2.zip
sudo ./aws/install
aws configure
```

## 🌍 Create cluster and node groups
```bash
eksctl create cluster \
  --name uche \
  --region eu-north-1 \
  --nodegroup-name standard-workers \
  --node-type c7i-flex.large \
  --nodes 3 \
  --nodes-min 1 \
  --nodes-max 3 \
  --managed \
  --node-volume-size 20 \
  --asg-access \
  --external-dns-access \
  --full-ecr-access \
  --appmesh-access \
  --alb-ingress-access

```

## ☸️ Configure kubeconfig for EKS

```bash
aws eks --region eu-north-1 update-kubeconfig --name uche
```

## 🔐 Associate IAM OIDC Provider with EKS

```bash
eksctl utils associate-iam-oidc-provider \
  --region eu-north-1 \
  --cluster uche \
  --approve
```

## 🛡️ Create IAM Service Account for EBS CSI Driver

```bash
eksctl create iamserviceaccount \
  --region ap-south-1 \
  --name ebs-csi-controller-sa \
  --namespace kube-system \
  --cluster uche \
  --attach-policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy \
  --approve \
  --override-existing-serviceaccounts
```

## 📦 Deploy Add-ons

### ✅ Install EBS CSI Driver

```bash
kubectl apply -k "github.com/kubernetes-sigs/aws-ebs-csi-driver/deploy/kubernetes/overlays/stable/ecr/?ref=release-1.11"
```

### 🌐 Install NGINX Ingress Controller

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/cloud/deploy.yaml
```

### 🔒 Install cert-manager

```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.12.0/cert-manager.yaml
```

### Helm Set Up

```bash
Install Helm
curl -fsSL https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
Downloading https://get.helm.sh/helm-v3.18.4-linux-amd64.tar.gz
```

### Set up Prometheus + Grafana via Helm 

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```

### Deploy prometheus chart in a new namespace “monitoring”

```bash
helm install monitoring prometheus-community/kube-prometheus-stack --namespace monitoring --create-namespace
```

### Verify Installation

```bash
kubectl get all -n monitoring
```

### Access Prometheus Dashboard

```bash
kubectl expose service prometheus-operated --type=NodePort --name=prometheus-nodeport -n monitoring --target-port=9090
```

### Get password to log into Grafana dashboard

```bash
kubectl get secret --namespace monitoring monitoring-grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo
```

### Access Grafana Dashboard

```bash
kubectl expose service monitoring-grafana --type=NodePort --target-port=3000 --name=grafana-nodeport -n monitoring
```

### Add Prometheus as a data source for Grafana

```bash
http://monitoring-kube-prometheus-prometheus.monitoring.svc.cluster.local:9090
```

### Import Dashboard in Grafana to visualise Prometheus metrics

```bash
Go to import dashboard
In 3662 
Click load
```

### Access Alertmanager Dashboard

```bash
kubectl expose service alertmanager-operated --type=NodePort --target-port=9093 --name=alertmanager-nodeport -n monitoring
```