# Cloud-Native Web Voting Application with Kubernetes

This cloud-native web application allows users to vote for their preferred programming language out of six choices: C#, Python, JavaScript, Go, Java, and NodeJS. The application is built using a mix of modern technologies and is designed to be accessible via the internet.

## Technical Stack

- **Frontend**: React and JavaScript
- **Backend and API**: Go (Golang) with MongoDB as the database backend

## Kubernetes Resources

This application leverages Kubernetes for deployment and management:

- **Namespace**: Isolated environments for different components
- **Secret**: Secure storage for sensitive information
- **Deployment**: Instructions for updates and scaling
- **Service**: Directs incoming traffic to appropriate instances
- **StatefulSet**: Manages MongoDB replica set
- **PersistentVolume and PersistentVolumeClaim**: Manages storage for data persistence

## Learning Opportunities

By working on this project, you will gain experience with:

1. **Containerization**: Using Docker to package applications
2. **Kubernetes Orchestration**: Managing, deploying, and scaling containerized applications
3. **Microservices Architecture**: Decoupling frontend and backend for independent scalability
4. **Database Replication**: Setting up a MongoDB replica set
5. **Security and Secrets Management**: Securing sensitive information using Kubernetes secrets
6. **Stateful Applications**: Deploying stateful applications within Kubernetes
7. **Persistent Storage**: Managing and provisioning persistent storage with Kubernetes

## Steps to Deploy

### Create EKS Cluster

1. **Create EKS Cluster with NodeGroup** (2 nodes of t2.medium instance type)
2. **Create EC2 Instance t2.micro** (Optional)

### IAM Role for EC2

```json
{
    "Version": "2012-10-17",
    "Statement": [{
        "Effect": "Allow",
        "Action": [
            "eks:DescribeCluster",
            "eks:ListClusters",
            "eks:DescribeNodegroup",
            "eks:ListNodegroups",
            "eks:ListUpdates",
            "eks:AccessKubernetesApi"
        ],
        "Resource": "*"
    }]
}
```

### Install Kubectl

```bash
curl -O https://s3.us-west-2.amazonaws.com/amazon-eks/1.24.11/2023-03-17/bin/linux/amd64/kubectl
chmod +x ./kubectl
sudo cp ./kubectl /usr/local/bin
export PATH=/usr/local/bin:$PATH
```

### Install AWS CLI

```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
```

### Configure EKS Cluster Context

```bash
aws eks update-kubeconfig --name EKS_CLUSTER_NAME --region us-west-2
```

### Check Nodes in Cluster

```bash
kubectl get nodes
```

### Create Namespace

```bash
kubectl create ns cloudchamp
kubectl config set-context --current --namespace cloudchamp
```

### MongoDB Setup

**Create MongoDB StatefulSet with Persistent Volumes**

```bash
kubectl apply -f mongo-statefulset.yaml
```

**Create MongoDB Service**

```bash
kubectl apply -f mongo-service.yaml
```

**Initialize MongoDB Replica Set**

Create a temporary network utils pod:

```bash
kubectl run --rm utils -it --image praqma/network-multitool -- bash
```

Within the utils pod shell, execute:

```bash
for i in {0..2}; do nslookup mongo-$i.mongo; done
```

Exit the utils container:

```bash
exit
```

Initialize the MongoDB Replica Set:

```bash
cat << EOF | kubectl exec -it mongo-0 -- mongo
rs.initiate();
sleep(2000);
rs.add("mongo-1.mongo:27017");
sleep(2000);
rs.add("mongo-2.mongo:27017");
sleep(2000);
cfg = rs.conf();
cfg.members[0].host = "mongo-0.mongo:27017";
rs.reconfig(cfg, {force: true});
sleep(5000);
EOF
```

Confirm the replica set:

```bash
kubectl exec -it mongo-0 -- mongo --eval "rs.status()" | grep "PRIMARY\|SECONDARY"
```

**Load Initial Data**

```bash
cat << EOF | kubectl exec -it mongo-0 -- mongo
use langdb;
db.languages.insert({"name" : "csharp", "codedetail" : { "usecase" : "system, web, server-side", "rank" : 5, "compiled" : false, "homepage" : "https://dotnet.microsoft.com/learn/csharp", "download" : "https://dotnet.microsoft.com/download/", "votes" : 0}});
db.languages.insert({"name" : "python", "codedetail" : { "usecase" : "system, web, server-side", "rank" : 3, "script" : false, "homepage" : "https://www.python.org/", "download" : "https://www.python.org/downloads/", "votes" : 0}});
db.languages.insert({"name" : "javascript", "codedetail" : { "usecase" : "web, client-side", "rank" : 7, "script" : false, "homepage" : "https://en.wikipedia.org/wiki/JavaScript", "download" : "n/a", "votes" : 0}});
db.languages.insert({"name" : "go", "codedetail" : { "usecase" : "system, web, server-side", "rank" : 12, "compiled" : true, "homepage" : "https://golang.org", "download" : "https://golang.org/dl/", "votes" : 0}});
db.languages.insert({"name" : "java", "codedetail" : { "usecase" : "system, web, server-side", "rank" : 1, "compiled" : true, "homepage" : "https://www.java.com/en/", "download" : "https://www.java.com/en/download/", "votes" : 0}});
db.languages.insert({"name" : "nodejs", "codedetail" : { "usecase" : "system, web, server-side", "rank" : 20, "script" : false, "homepage" : "https://nodejs.org/en/", "download" : "https://nodejs.org/en/download/", "votes" : 0}});
db.languages.find().pretty();
EOF
```

**Create Mongo Secret**

```bash
kubectl apply -f mongo-secret.yaml
```

### API Setup

**Create GO API Deployment**

```bash
kubectl apply -f api-deployment.yaml
```

**Expose API Deployment**

```bash
kubectl expose deploy api --name=api --type=LoadBalancer --port=80 --target-port=8080
```

**Set Environment Variable**

```bash
{
API_ELB_PUBLIC_FQDN=$(kubectl get svc api -ojsonpath="{.status.loadBalancer.ingress[0].hostname}")
until nslookup $API_ELB_PUBLIC_FQDN >/dev/null 2>&1; do sleep 2 && echo waiting for DNS to propagate...; done
curl $API_ELB_PUBLIC_FQDN/ok
echo
}
```

**Test API Endpoints**

```bash
curl -s $API_ELB_PUBLIC_FQDN/languages | jq .
curl -s $API_ELB_PUBLIC_FQDN/languages/go | jq .
curl -s $API_ELB_PUBLIC_FQDN/languages/java | jq .
curl -s $API_ELB_PUBLIC_FQDN/languages/nodejs | jq .
```

### Frontend Setup

**Create Frontend Deployment**

```bash
kubectl apply -f frontend-deployment.yaml
```

**Expose Frontend Deployment**

```bash
kubectl expose deploy frontend --name=frontend --type=LoadBalancer --port=80 --target-port=8080
```

**Confirm Frontend ELB**

```bash
{
FRONTEND_ELB_PUBLIC_FQDN=$(kubectl get svc frontend -ojsonpath="{.status.loadBalancer.ingress[0].hostname}")
until nslookup $FRONTEND_ELB_PUBLIC_FQDN >/dev/null 2>&1; do sleep 2 && echo waiting for DNS to propagate...; done
curl -I $FRONTEND_ELB_PUBLIC_FQDN
}
```

**Generate Frontend URL**

```bash
echo http://$FRONTEND_ELB_PUBLIC_FQDN
```

### Test the Application

1. **Access the Application**: Use your browser to visit the frontend URL.
2. **Vote**: Click on the **+1** buttons to vote for your preferred language.
3. **Query MongoDB**: Confirm the vote data is updated:

```bash
kubectl exec -it mongo-0 -- mongo langdb --eval "db.languages.find().pretty()"
```

## Summary

In this project, you deployed a cloud-native application into EKS. You tested the application via your browser and confirmed that your activities generated data captured within the MongoDB ReplicaSet backend in the cluster.

## Repository

[GitHub Repository](https://github.com/harishsemwal/Kubernetes-Cloud-Native-Voting-Application.git)

---
