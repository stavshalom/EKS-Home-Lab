# Steps to Complete the EKS Monitoring Lab

---

## Table of Contents
1. [Understanding the Objectives](#1-understanding-the-objectives)
2. [Lessons Learned and Troubleshooting Insights](#2-lessons-learned-and-troubleshooting-insights)
   - [Key Takeaways](#key-takeaways)
   - [Troubleshooting Highlights](#troubleshooting-highlights)
3. [Learning Resources](#3-learning-resources)
4. [Step 1: Creating a New EKS Cluster](#4-step-1-creating-a-new-eks-cluster)
5. [Step 2: Installing Prometheus and Grafana](#5-step-2-installing-prometheus-and-grafana)
   - [Installing Prometheus](#installing-prometheus)
   - [Handling Pending Pods](#handling-pending-pods)
   - [Installing the AWS EBS CSI Driver](#installing-the-aws-ebs-csi-driver)
   - [Creating a StorageClass for the EBS CSI Driver](#creating-a-storageclass-for-the-ebs-csi-driver)
   - [Resolving Pending Pods](#resolving-pending-pods)
6. [Accessing Prometheus and Grafana](#6-accessing-prometheus-and-grafana)
7. [Conclusion](#7-conclusion)

---

## 1. Understanding the Objectives

This lab focuses on building a robust monitoring system for an EKS cluster. The primary goals are:
- **Performance Tracking**: Monitor Pods and Nodes for CPU, memory, and storage usage metrics.
- **Workload Analysis**: Understand cluster behavior under different loads.
- **Log Management**: Enable effective troubleshooting by centralizing and analyzing logs.

---

## 2. Lessons Learned and Troubleshooting Insights

### Key Takeaways
1. **Hands-on Practice**: Gained practical experience in Kubernetes resource management, especially PVCs and PVs.
2. **AWS Integration**: Learned how cloud-native storage solutions, like AWS EBS CSI Driver, integrate with Kubernetes.
3. **Efficient Resource Allocation**: Understood the limitations of small instances like `t2.micro` and the importance of resource optimization.

### Troubleshooting Highlights
1. **PVC Binding Issues**:
   - **Problem**: PVCs remained in a `Pending` state due to missing or mismatched StorageClass definitions.
   - **Solution**: Created and applied correct `StorageClass` configurations, ensuring compatibility with PVs.

2. **Pod Scheduling Problems**:
   - **Problem**: Prometheus Pods were stuck in `Pending` due to insufficient node resources.
   - **Solution**: Adjusted resource requests and limits in Helm Chart values, optimizing them for the cluster's capacity.

3. **AWS Permissions**:
   - **Problem**: EBS CSI Driver required additional IAM permissions.
   - **Solution**: Attached the necessary IAM policy (`AmazonEKS_EBS_CSI_Driver`) to the Node Group's IAM role.

4. **Grafana Access Challenges**:
   - **Problem**: Difficulty accessing Grafana dashboards initially.
   - **Solution**: Resolved by performing Port Forwarding and retrieving admin credentials from Kubernetes secrets.

---

## 3. Learning Resources

### Prometheus Basics
[Quick YouTube Guide to Prometheus](https://www.youtube.com/watch?v=h4Sl21AKiDg&t=31s)

### Helm Basics in Kubernetes
[Using Helm for Kubernetes](https://www.youtube.com/watch?v=-ykwb1d0DXU)

### Grafana for Beginners
[Beginner's Playlist for Grafana](https://www.youtube.com/watch?v=TQur9GJHIIQ&list=PLDGkOdUX1Ujo27m6qiTPPCpFHVfyKq9jT)

---

## 4. Step 1: Creating a New EKS Cluster

Create an EKS cluster with three `t3.medium` nodes:
```bash
eksctl create cluster --name eks-stav --region eu-west-1 --nodes 3 --node-type t3.medium --managed
```
---

## 5. Step 2: Installing Prometheus and Grafana

### Installing Prometheus

1. **Install Helm**:
   ```bash
   curl https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 | bash
   ```

2. **Add the Prometheus Community Repository**:
   ```bash
   helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
   helm repo update
   ```

3. **Install Prometheus**:
   ```bash
   helm install prometheus prometheus-community/prometheus --namespace monitoring --create-namespace
   ```

4. **Verify Prometheus is Running**:
   ```bash
   kubectl get pods -n monitoring
   ```

![Prometheus Pods](https://github.com/user-attachments/assets/f37a378f-51ba-44d8-bcfa-95df7b633021)

---

### Handling Pending Pods

Some Prometheus Pods, such as `prometheus-alertmanager-0`, may enter a **`Pending`** state due to PVC issues. Resolve these by ensuring proper PV and StorageClass configurations.

#### Investigating PVC Issues
1. **List Pending PVCs**:
   ```bash
   kubectl get pvc -n monitoring
   ```
2. **Describe a Pending PVC**:
   ```bash
   kubectl describe pvc <pvc-name> -n monitoring
   ```

---

### Installing the AWS EBS CSI Driver

1. **Add the EBS CSI Driver Repository**:
   ```bash
   helm repo add aws-ebs-csi-driver https://kubernetes-sigs.github.io/aws-ebs-csi-driver
   helm repo update
   ```

2. **Install the Driver**:
   ```bash
   helm install aws-ebs-csi-driver aws-ebs-csi-driver/aws-ebs-csi-driver \
   --namespace kube-system \
   --set controller.serviceAccount.create=true \
   --set controller.serviceAccount.name=ebs-csi-controller-sa \
   --set region=eu-west-1
   ```

3. **Verify the Installation**:
   ```bash
   kubectl get pods -n kube-system -l app.kubernetes.io/name=aws-ebs-csi-driver
   ```

---

### Creating a StorageClass for the EBS CSI Driver

1. Create a `StorageClass` YAML file (e.g., `ebs-storageclass.yaml`):
   ```yaml
   apiVersion: storage.k8s.io/v1
   kind: StorageClass
   metadata:
     name: ebs-sc
   provisioner: ebs.csi.aws.com
     parameters:
       type: gp2
       fsType: ext4
   reclaimPolicy: Retain
   volumeBindingMode: WaitForFirstConsumer
   ```
   Apply the `StorageClass`:
   ```bash
   kubectl apply -f ebs-storageclass.yaml
   ```

---

### Resolving Pending Pods

#### Handling `prometheus-server` Pod
1. **Create a PersistentVolume (PV)**:
   Create a file named `prometheus-server-pv.yaml`:
   ```yaml
   apiVersion: v1
   kind: PersistentVolume
   metadata:
     name: prometheus-server
   spec:
     capacity:
       storage: 10Gi
     volumeMode: Filesystem
     accessModes:
       - ReadWriteOnce
     persistentVolumeReclaimPolicy: Retain
     storageClassName: gp2
     csi:
        driver: ebs.csi.aws.com
        volumeHandle: <your-ebs-volume-id>
        fsType: ext4
      nodeAffinity:
        required:
          nodeSelectorTerms:
            - matchExpressions:
                - key: topology.kubernetes.io/zone
                  operator: In
                  values:
                    - eu-west-1a
   ```
   Apply the PV:
   ```bash
   kubectl apply -f prometheus-server-pv.yaml
   ```

2. **Delete and Recreate the PVC**:
   ```bash
   kubectl delete pvc prometheus-server -n monitoring
   ```
   Create a file named `prometheus-server-pvc.yaml`:
   ```yaml
   apiVersion: v1
   kind: PersistentVolumeClaim
   metadata:
     name: prometheus-server
     namespace: monitoring
   spec:
     accessModes:
       - ReadWriteOnce
     resources:
       requests:
         storage: 10Gi
     storageClassName: gp2
   ```
   Apply the PVC:
   ```bash
   kubectl apply -f prometheus-server-pvc.yaml
   ```

3. **Verify the PVC and PV are Bound**:
   ```bash
   kubectl get pvc -n monitoring
   kubectl get pv
   ```
![image](https://github.com/user-attachments/assets/460f6148-c7ae-4e80-8bf1-337dbd3b5cca)

---

#### Handling `prometheus-alertmanager-0` Pod
1. **Create a New StorageClass with Immediate Binding**:
   ```yaml
   apiVersion: storage.k8s.io/v1
   kind: StorageClass
   metadata:
     name: gp2-immediate
   provisioner: ebs.csi.aws.com
   parameters:
     type: gp2
   reclaimPolicy: Retain
   volumeBindingMode: Immediate
   ```
   Apply the StorageClass:
   ```bash
   kubectl apply -f gp2-immediate.yaml
   ```

2. **Create a PersistentVolume for Alertmanager**:
   ```yaml
   apiVersion: v1
   kind: PersistentVolume
   metadata:
     name: prometheus-alertmanager-pv
   spec:
     capacity:
       storage: 10Gi
     accessModes:
       - ReadWriteOnce
     storageClassName: gp2-immediate
     awsElasticBlockStore:
       volumeID: <your-ebs-volume-id>
       fsType: ext4
   ```
   Apply the PV:
   ```bash
   kubectl apply -f prometheus-alertmanager-pv.yaml
   ```

3. **Recreate the StatefulSet**:
   Delete the old StatefulSet:
   ```bash
   kubectl delete statefulset prometheus-alertmanager -n monitoring
   ```
   Create a new YAML file for the StatefulSet (`statefulset-alertmanager.yaml`) and apply it:
   ```bash
   kubectl apply -f statefulset-alertmanager.yaml
   ```

4. **Verify the Pod is Running**:
   ```bash
   kubectl get pods -n monitoring
   ```

---

## 6. Accessing Prometheus and Grafana

### Prometheus
1. Find the Prometheus Service:
   ```bash
   kubectl get services -n monitoring
   ```
2. Perform Port Forwarding:
   ```bash
   kubectl port-forward -n monitoring svc/prometheus-server 9090:80
   ```
3. Access Prometheus at:
   [http://localhost:9090](http://localhost:9090)

### Grafana
1. Add the Grafana Repository:
   ```bash
   helm repo add grafana https://grafana.github.io/helm-charts
   helm repo update
   ```
2. Install Grafana:
   ```bash
   helm install grafana grafana/grafana --namespace monitoring
   ```
3. Retrieve the Admin Password:
   ```bash
   kubectl get secret --namespace monitoring grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo
   ```
4. Perform Port Forwarding to Access Grafana:
   ```bash
   kubectl port-forward -n monitoring svc/grafana 3000:80
   ```
5. Access Grafana at:
   [http://localhost:3000](http://localhost:3000)

---

## 7. Conclusion

This lab was a comprehensive exercise in setting up monitoring for Kubernetes clusters using Prometheus and Grafana. It reinforced key concepts in Kubernetes resource management, cloud-native storage solutions, and troubleshooting techniques. Future enhancements could include automating the entire setup using Helm charts and scripts to streamline the process further.
