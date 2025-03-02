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
7. [Cleanup Gg All Resources in an EKS Cuide](#7-cleanup-gg-all-resources-in-an-eks-guide)
8. [Conclusion](#8-conclusion)

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
   ![image](https://github.com/user-attachments/assets/bf1b92ca-2669-4e3f-8104-d6e12b61b514)


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
   ![image](https://github.com/user-attachments/assets/13fc0ca3-a12e-4096-b18f-cf2fbbde3556)

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
   ![image](https://github.com/user-attachments/assets/d17d274a-fb35-42b1-9785-576612a33653)

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
   ![image](https://github.com/user-attachments/assets/fad5c6de-bda3-4d68-a967-211b8820cbcc)

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

   ![image](https://github.com/user-attachments/assets/d46637b9-580c-48b8-9c45-f97c5b15b324)


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
## 7. Cleanup Gg All Resources in an EKS Cuide

## Step 1: Deleting All Resources in Namespaces

1. Delete all resources in all namespaces:
   ```bash
   kubectl delete all --all --all-namespaces
   ```
2. Delete ConfigMaps and Secrets in all namespaces:
   ```bash
   kubectl delete configmaps --all --all-namespaces
   kubectl delete secrets --all --all-namespaces
   ```

---

## Step 2: Deleting PersistentVolumeClaims and PersistentVolumes

1. Delete all PVCs:
   ```bash
   kubectl delete pvc --all --all-namespaces
   ```
2. Delete all PVs:
   ```bash
   kubectl delete pv --all
   ```

---

## Step 3: Deleting StorageClasses

1. Delete all StorageClasses (if any exist):
   ```bash
   kubectl delete storageclass --all
   ```

---

## Step 4: Deleting Custom Namespaces

1. If you created custom namespaces, delete them:
   ```bash
   kubectl delete namespace <namespace-name>
   ```
2. To delete all custom namespaces:
   ```bash
   kubectl get namespaces | grep -v "kube-system\|kube-public\|default" | awk '{print $1}' | xargs kubectl delete namespace
   ```

---

## Step 5: Deleting the AWS EBS CSI Driver

1. Uninstall the AWS EBS CSI Driver:
   ```bash
   kubectl delete -k "github.com/kubernetes-sigs/aws-ebs-csi-driver/deploy/kubernetes/overlays/stable/ecr/?ref=release-1.38"
   ```

---

## Step 6: Deleting Nodes and the EKS Cluster (Optional)

### Deleting the Cluster

1. If you used `eksctl` to create the cluster, delete it:
   ```bash
   eksctl delete cluster --name eks-stav
   ```
2. If the cluster was created manually (e.g., via CloudFormation), delete the stack:
   ```bash
   aws cloudformation delete-stack --stack-name <stack-name>
   ```

### Deleting Nodes (if not automatically removed)

1. Identify the EC2 instances associated with the cluster:
   ```bash
   aws ec2 describe-instances --filters "Name=tag:eks:cluster-name,Values=<cluster-name>" --query "Reservations[].Instances[].InstanceId"
   ```
2. Terminate the instances:
   ```bash
   aws ec2 terminate-instances --instance-ids <instance-id-1> <instance-id-2>
   ```

---

## Step 7: Final Cleanup

1. Check for orphaned Load Balancers:
   ```bash
   aws elb describe-load-balancers --query "LoadBalancerDescriptions[].LoadBalancerName"
   ```
   Delete any orphaned Load Balancers:
   ```bash
   aws elb delete-load-balancer --load-balancer-name <load-balancer-name>
   ```

2. Delete orphaned Security Groups:
   ```bash
   aws ec2 delete-security-group --group-id <security-group-id>
   ```

3. Delete orphaned Volumes:
   ```bash
   aws ec2 delete-volume --volume-id <volume-id>
   ```

---

## Step 8: Verifying Resource Deletion

1. Verify no active resources exist in EKS:
   ```bash
   kubectl get all --all-namespaces
   ```

2. Confirm no clusters are present:
   ```bash
   eksctl get cluster
   ```

3. Use the AWS Management Console to ensure no active resources remain in EC2, S3, or other services.

---

By following these steps, you ensure that all resources associated with your EKS cluster are properly deleted, minimizing costs and avoiding orphaned resources in your AWS account.

---
## 8. Conclusion

This lab was a comprehensive exercise in setting up monitoring for Kubernetes clusters using Prometheus and Grafana. It reinforced key concepts in Kubernetes resource management, cloud-native storage solutions, and troubleshooting techniques. Future enhancements could include automating the entire setup using Helm charts and scripts to streamline the process further.
