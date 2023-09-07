# aws-jaeger-kiali-integration


# Create AWS EC2 Instance and Integrate Jaeger and Kiali


## Important Concepts

### Jaeger
Used for distributed tracing

### Kiali
It is integrated with Istio and shows the hops(big picture) of routing & networking

### Ingress
Load Balancer helps for internal routing and communication (layer 2)

### Istio
Service Mesh component which helps in internal routing (Top layer)


## Process

### 1. Create IAM user (Authentication)

- Search **IAM** -> Click Users -> Create User
- Click on the created User -> Security credentials -> Access Keys -> Create Access Key -> CLI -> Create User Access Key -> Download Access Key CSV
- Click on the created User -> Security credentials -> Console Sign In -> Enable -> Download User Credentials

### 2. IAM user (Authorization)

- Search **IAM** -> Click Users -> Select the User -> Permissions -> Add Permissions -> Attach Policies directly -> "AdministratorAccess"
- Sign In from IAM
- Change Region to us-west-1

### 3. Create the AWS EC2 Linux AMI Instance
- Name the Instance -> Select Amazon Linux -> AMI as **Amazon Linux 2023 AMI**
- Instance Type - t2.medium (requirement for EKS)
- Create a New Key-Pair and Select It
- Launch EC2 Instance
- Once created -> click the check box against the EC2 Instance -> Click Connect -> **sudo su**

### 4. Install kubectl (We will be accessing the PODS and resources of k8s)
- Hit and check kubectl
```
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"

```
- Download SHA256
```
curl -LO "https://dl.k8s.io/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl.sha256"
```
- Check Key and Kubectl is OK
```
echo "$(cat kubectl.sha256) kubectl" | sha256sum --check
```
- Install kubectl
```
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
```

### 5. Install eskctl (To create cluster)
- Hit the end point to get the data -> unzip it in \tmp folder
```
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
```
- Move to \bin folder
```
sudo mv /tmp/eksctl /usr/bin
```
- To check version
```
eksctl version
```

## 6. Add IAM Role to EC2 (so that EC2 access the EKS)

- Go to IAM Dashboard -> Select Roles -> Create Role -> Select AWS service -> Select EC2 -> Next -> Give Administrator Access -> Give Role Name -> Create Role
- Go to EC2 Instance -> Check the Instance -> Select Actions -> Security -> Modify IAM -> Associate the IAM Role

## 7. Create Cluster
Create EKS Cluster
```
eksctl create cluster --name=eksdemo1 --region=us-west-1 --zones=us-west-1c,us-west-1a --without-nodegroup
```

### 8. Add Open ID Connect(OIDC, approval to k8s to access other resources)
```
eksctl utils associate-iam-oidc-provider --region us-west-1 --cluster eksdemo1 --approve
```

### 9. Add nodes
```
eksctl create nodegroup --cluster=eksdemo1 --region=us-west-1 --name=eksdemo-ng-public --node-type=t2.medium --nodes=2 --nodes-min=2 --nodes-max=4 --node-volume-size=10 --ssh-access --ssh-public-key=aws-jaeger-kiali-integration-key-pair --managed --asg-access --external-dns-access --full-ecr-access --appmesh-access --alb-ingress-access
```