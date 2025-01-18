# Steps to Complete the EKS Monitoring Lab

## Understanding the Objectives

Before starting, it's important to define the monitoring goals:
- Track the performance of Pods and Nodes (CPU, memory, storage, etc.).
- Analyze the cluster's workload.
- Manage logs for troubleshooting.

## Learning Resources

### Prometheus Basics:
A quick YouTube video to understand Prometheus:
[https://www.youtube.com/watch?v=h4Sl21AKiDg&t=31s](https://www.youtube.com/watch?v=h4Sl21AKiDg&t=31s)

### Helm Basics in Kubernetes:
An explanatory video on using Helm:
[https://www.youtube.com/watch?v=-ykwb1d0DXU](https://www.youtube.com/watch?v=-ykwb1d0DXU)

### Grafana for Beginners:
A playlist of 12 videos for beginners:
[https://www.youtube.com/watch?v=TQur9GJHIIQ&list=PLDGkOdUX1Ujo27m6qiTPPCpFHVfyKq9jT](https://www.youtube.com/watch?v=TQur9GJHIIQ&list=PLDGkOdUX1Ujo27m6qiTPPCpFHVfyKq9jT)

---

## Step 1: Creating a New EKS Cluster

Create a new EKS cluster with 3 nodes of type `t3.medium`:
```bash
eksctl create cluster --name eks-stav --region eu-west-1 --nodes 3 --node-type t3.medium --managed
```

---

## Step 2: Installing Prometheus and Grafana

Prometheus and Grafana are the most common tools for Kubernetes monitoring.

### Installing Prometheus

1. **Install Helm**:
   ```bash
   curl https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 | bash
   ```
   ![image](https://github.com/user-attachments/assets/66a415f6-4054-4fd2-9a17-87962126c4ca)


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
   ![image](https://github.com/user-attachments/assets/f37a378f-51ba-44d8-bcfa-95df7b633021)


---

### Handling Pending Pods

Some Prometheus Pods, such as `prometheus-alertmanager-0` and `prometheus-server`, may enter a **`Pending`** state. This usually occurs due to issues with PersistentVolumeClaims (PVCs).

#### Investigate PVC Issues
1. **List Pending PVCs**:
   ```bash
   kubectl get pvc -n monitoring
   ```
2. **Describe a Pending PVC**:
   ```bash
   kubectl describe pvc <pvc-name> -n monitoring
   ```
3. **Check Available PersistentVolumes (PVs)**:
   ```bash
   kubectl get pv
   ```
   If no PVs are available, you will need to create them manually.

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

3. **Verify the Driver Installation**:
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

2. Apply the `StorageClass`:
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

### Accessing Prometheus
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

---

![image](https://github.com/user-attachments/assets/1a5044bb-a8e9-4909-86bd-c46b1750453d)

### Step 3: Installing Grafana

1. **Add the Grafana Repository**:
   ```bash
   helm repo add grafana https://grafana.github.io/helm-charts
   helm repo update
   ```

2. **Install Grafana**:
   ```bash
   helm install grafana grafana/grafana --namespace monitoring
   ```

3. **Find the Admin Password**:
   ```bash
   kubectl get secret --namespace monitoring grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo
   ```

4. **Access Grafana**:
   Perform
