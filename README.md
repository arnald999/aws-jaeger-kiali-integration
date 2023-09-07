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
- Important components of Kubernetes(default namespace - kube-system)
```
kubectl -n kube-system get pods
```

### 10. Install Istio
```
curl -L https://istio.io/downloadIstio | ISTIO_VERSION=1.18.1 TARGET_ARCH=x86_64 sh -
```
```
cd istio-1.18.1
```

Important files - (samples, manifests(profiles, pod related files))

### 11. Set the Path
- export PATH=$PWD/bin:$PATH


### 12. For Connection Error
```
aws eks update-kubeconfig --name eksdemo1 --region us-west-1
```


### 13. INSTALL THE ISTIO WITH DEMO PROFILE (Profile an be demo, prod, etc.)
```
istioctl install --set profile=demo -y
```

### 14. Install Application from Github
```
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.18/samples/bookinfo/platform/kube/bookinfo.yaml
```
Check if things are running
```
- kubectl get services
- kubectl get pods

- kubectl exec "$(kubectl get pod -l app=ratings -o jsonpath='{.items[0].metadata.name}')" -c ratings -- curl -sS productpage:9080/productpage | grep -o "<title>.*</title>"

```


### 15. TO INJECT ISTIO AS INIT CONTAINER [NOW 2 PODS WILL RUN ]
- kubectl label namespace default istio-injection=enabled
- istioctl analyze
- Delete all pods
kubectl delete pod <pod_name>

### 16. cd samples/bookinfo/networking/
- kubectl apply -f bookinfo-gateway.yaml

### 17. kubectl get vs
- kubectl get gateway
- kubectl get svc istio-ingressgateway -n istio-system

### 18. Set the ingress IP and ports:

```
export INGRESS_HOST=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
export INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="http2")].port}')
export SECURE_INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="https")].port}')

```

- echo $SECURE_INGRESS_PORT

### 19. Forming Gateway URL(DNS from Load Balancer)
- export INGRESS_HOST=ae8e08fbd79ab46c9be1ecf51cbf532a-1481031991.us-west-1.elb.amazonaws.com
- export GATEWAY_URL=$INGRESS_HOST:$INGRESS_PORT
- echo $GATEWAY_URL


### 20. Hit the below URL
- echo "http://$GATEWAY_URL/productpage"

### 21. KIALI DASHOBAORD [ ALL TOOLS INSTALLATION ]
```
cd istio-1.18.1/samples/addons

kubectl apply -f .
```

### 22. DO PORT FORWARD
```
kubectl port-forward --address 0.0.0.0 svc/kiali 9008:20001 -n istio-system
```


### 23. OPEN THE Security Group TO ALL TRAFFIC
- EC2 Instance -> Security -> Security group -> Type (All Traffic) -> Source (Anywhere IPv4)

### 24. Open Kiali Dashboard
- http://Public IPv4:FORWARDED-PORT/kiali


### 25. For JAEGER
```
kubectl port-forward --address 0.0.0.0 svc/tracing 8008:80 -n istio-system
```

### 26. Open Jaeger Dashboard
- http://Public IPv4:FORWARDED-PORT/jaeger

### 27. Delete node
```
eksctl delete nodegroup --cluster=eksdemo1 --region=us-west-1 --name=eksdemo-ng-public
```

### 28. Delete cluster
