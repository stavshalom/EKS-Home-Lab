# Steps for Home Lab Task 1: EKS Cluster Setup and Connection

---

## 1. Setting Up AWS Free Tier Account
- Create a personal AWS account with Free Tier access.

## 2. Configuring IAM User
- Create a new IAM user with **AdministratorAccess** permissions.
- Generate **Access Keys** (Access Key ID and Secret Access Key) for the user.
- Save the keys securely as they will be used to connect to AWS from the Ubuntu server.

---

## 3. Preparing Virtual Machine on Local Laptop
1. **Install VirtualBox**:
   - Download and install VirtualBox on your laptop.

2. **Download Ubuntu Server ISO**:
   - Download the ISO for Ubuntu Server 24.04.1 LTS using the official Ubuntu website.

3. **Set Up Virtual Machine**:
   - Create a virtual machine in VirtualBox and allocate appropriate resources (e.g., 2 CPUs, 4GB RAM).

4. **Install Ubuntu Server**:
   - Use the downloaded ISO to install Ubuntu Server on the virtual machine.

5. **Configure Port Forwarding**:
   - Forward SSH traffic from the host machine to the VM.
   - Example: Map port `2222` on the host to port `22` on the guest.

6. **Connect via SSH**:
   - Use PuTTY or another SSH client to connect to the Ubuntu VM from the host machine.

---

## 4. Installing AWS CLI on Ubuntu Server
1. **Install AWS CLI v2**:
   ```bash
   curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
   unzip awscliv2.zip
   sudo ./aws/install
   ```

2. **Configure AWS CLI**:
   - Run the configuration command:
     ```bash
     aws configure
     ```
   - Provide the **Access Key ID** and **Secret Access Key** generated in Step 2.
   - Specify the default region (e.g., `eu-west-1`).

---

## 5. Installing kubectl
1. **Update System Packages**:
   ```bash
   sudo apt-get update
   sudo apt-get upgrade -y
   ```

2. **Manually Install kubectl**:
   - Download the kubectl binary:
     ```bash
     curl -LO https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl
     ```
   - Make the file executable:
     ```bash
     chmod +x ./kubectl
     ```
   - Move it to a system-wide path:
     ```bash
     sudo mv ./kubectl /usr/local/bin/kubectl
     ```
   - Verify the installation:
     ```bash
     kubectl version --client
     ```

---

## 6. Installing eksctl
- Install `eksctl` to simplify EKS cluster creation:
  ```bash
  sudo curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | sudo tar xz -C /usr/local/bin
  ```

---

## 7. Creating an EKS Cluster
1. **Run the Cluster Creation Command**:
   ```bash
   eksctl create cluster --name eks-stav --region eu-west-1 --nodes 2 --node-type t2.micro --managed
   ```
   - **Cluster Name**: `eks-stav`  
   - **Region**: `eu-west-1` (Ireland)  
   - **Node Type**: `t2.micro` (Free Tier eligible for up to 750 hours/month for the first year).

2. **Enable OIDC Provider**:
   - Associate an OIDC provider with the cluster:
     ```bash
     eksctl utils associate-iam-oidc-provider --region eu-west-1 --cluster eks-stav --approve
     ```

3. **Verify Cluster Creation**:
   - Check the cluster status using `eksctl` or `kubectl`.

---

## 8. Connecting to the EKS Cluster
1. **Update kubeconfig**:
   ```bash
   aws eks update-kubeconfig --region eu-west-1 --name eks-stav
   ```
2. **Verify the Connection**:
   ```bash
   kubectl get namespace
   ```
   ![image](https://github.com/user-attachments/assets/ee210189-1f6e-410f-a51f-50765adb14f8)


---

## 9. Scaling Down to Reduce Costs
1. **Set Minimum Node Count to 0**:
   ```bash
   eksctl scale nodegroup --cluster=eks-stav --name=ng-4b7f3173 --nodes-min=0
   ```

2. **Set Desired Node Count to 0**:
   ```bash
   eksctl scale nodegroup --cluster=eks-stav --name=ng-4b7f3173 --nodes=0
   ```

3. **Verify the Scaling**:
   ```bash
   kubectl get nodes
   ```

---

## 10. Cleaning Up Resources
1. **Delete the Node Group**:
   ```bash
   eksctl delete nodegroup --cluster=eks-stav --name=ng-4b7f3173
   ```

2. **Delete the Cluster**:
   ```bash
   eksctl delete cluster --name eks-stav
   ```

---

## Key Learnings from the Process
1. **Hands-On with Kubernetes and AWS:**
   - Learned to set up and manage an EKS cluster, including node scaling and kubeconfig updates.
2. **Cost Optimization:**
   - Understood how to minimize cloud costs by scaling down resources and leveraging AWS Free Tier.
3. **Troubleshooting and Problem-Solving:**
   - Addressed challenges such as OIDC setup and manual `kubectl` installation.
4. **Improved Automation Skills:**
   - Used `eksctl` for streamlined cluster management, showcasing the power of automation.

---

