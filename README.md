# BankApp CD Pipeline - Kubernetes Deployment

## 🚀 Overview
Continuous Deployment pipeline for BankApp using Jenkins and Kubernetes (AWS EKS).

## 📋 Prerequisites
- AWS EKS Cluster running
- kubectl installed and configured
- AWS CLI installed
- Jenkins with necessary plugins
- Docker Hub account

---

## 🏗️ Infrastructure Setup

### 1. Install kubectl
```bash
curl -LO https://dl.k8s.io/release/v1.31.0/bin/linux/amd64/kubectl
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
kubectl version --client
```

### 2. Install AWS CLI
```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
aws --version
```

### 3. Configure AWS Credentials
```bash
aws configure
# Enter:
# - AWS Access Key ID
# - AWS Secret Access Key
# - Default region: ap-south-1
# - Default output format: json
```

### 4. Configure kubectl for EKS
```bash
aws eks update-kubeconfig --region ap-south-1 --name YOUR_CLUSTER_NAME
kubectl get nodes
```

---

## 📦 Kubernetes Manifests

### Directory Structure
```
BankApp-CD/
├── database/
│   └── database-ds.yml
├── bankapp/
│   └── bankapp-ds.yml
├── Jenkinsfile
└── README.md
```

### Database Deployment (`database/database-ds.yml`)
```yaml
---
# MySQL Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql
spec:
  selector:
    matchLabels:
      app: mysql
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - image: mysql:8
        name: mysql
        env:
        - name: MYSQL_ROOT_PASSWORD
          value: "Test@123"
        - name: MYSQL_DATABASE
          value: "bankappdb"
        ports:
        - containerPort: 3306
          name: mysql
---
# MySQL Service
apiVersion: v1
kind: Service
metadata:
  name: mysql-service
spec:
  ports:
  - port: 3306
  selector:
    app: mysql
```

### Application Deployment (`bankapp/bankapp-ds.yml`)
```yaml
---
# Java Application Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: bankapp
spec:
  replicas: 2
  revisionHistoryLimit: 1
  selector:
    matchLabels:
      app: bankapp
  template:
    metadata:
      labels:
        app: bankapp
    spec:
      containers:
      - name: bankapp
        image: YOUR_USERNAME/bankapp:v24
        ports:
        - containerPort: 8080
        env:
        - name: SPRING_DATASOURCE_URL
          value: jdbc:mysql://mysql-service:3306/bankappdb?useSSL=false&serverTimezone=UTC&allowPublicKeyRetrieval=true
        - name: SPRING_DATASOURCE_USERNAME
          value: root
        - name: SPRING_DATASOURCE_PASSWORD
          value: Test@123
---
# Bank Application Service
apiVersion: v1
kind: Service
metadata:
  name: bankapp-service
spec:
  type: LoadBalancer
  ports:
  - port: 80
    targetPort: 8080
  selector:
    app: bankapp
```

---

## 🔧 Jenkins CD Pipeline Setup

### Step 1: Create CD Pipeline Job

1. Go to Jenkins Dashboard
2. Click "New Item"
3. Enter name: `CD`
4. Select "Pipeline"
5. Click "OK"

### Step 2: Configure Pipeline

1. **General**:
   - Description: "CD Pipeline for BankApp Kubernetes Deployment"
   - Discard old builds: Keep last 5 builds

2. **Pipeline**:
   - Definition: Pipeline script from SCM
   - SCM: Git
   - Repository URL: `https://github.com/YOUR_USERNAME/BankApp-CD.git`
   - Branch: `*/main`
   - Script Path: `Jenkinsfile`

### Step 3: Configure Jenkins User for Kubernetes

```bash
# Copy AWS credentials to Jenkins user
sudo -u jenkins mkdir -p /var/lib/jenkins/.aws
sudo cp ~/.aws/credentials /var/lib/jenkins/.aws/
sudo cp ~/.aws/config /var/lib/jenkins/.aws/
sudo chown -R jenkins:jenkins /var/lib/jenkins/.aws

# Configure kubectl for Jenkins user
sudo -u jenkins aws eks update-kubeconfig --region ap-south-1 --name YOUR_CLUSTER_NAME
```

---

## 📝 Jenkinsfile

```groovy
pipeline {
    agent any
    
    environment {
        GIT_REPO = 'https://github.com/YOUR_USERNAME/BankApp-CD.git'
        GIT_BRANCH = 'main'
        EKS_CLUSTER = 'YOUR_CLUSTER_NAME'
        AWS_REGION = 'ap-south-1'
    }
    
    stages {
        stage('Clean Workspace') {
            steps {
                cleanWs()
            }
        }
        
        stage('Checkout CD Repo') {
            steps {
                git branch: "${GIT_BRANCH}", url: "${GIT_REPO}"
            }
        }
        
        stage('Update Kubeconfig') {
            steps {
                sh """
                    aws eks update-kubeconfig --region ${AWS_REGION} --name ${EKS_CLUSTER}
                    kubectl version --client
                """
            }
        }
        
        stage('Deploy Database') {
            steps {
                sh """
                    kubectl apply -f database/database-ds.yml
                    kubectl rollout status deployment/mysql -n default --timeout=300s
                """
            }
        }
        
        stage('Deploy Application') {
            steps {
                sh """
                    kubectl apply -f bankapp/bankapp-ds.yml
                    kubectl rollout status deployment/bankapp -n default --timeout=300s
                """
            }
        }
        
        stage('Verify Deployment') {
            steps {
                sh """
                    echo "=== Pods Status ==="
                    kubectl get pods -l app=bankapp
                    kubectl get pods -l app=mysql
                    
                    echo "=== Services ==="
                    kubectl get svc bankapp-service
                    kubectl get svc mysql-service
                    
                    echo "=== Deployment Status ==="
                    kubectl get deployment bankapp
                    kubectl get deployment mysql
                """
            }
        }
        
        stage('Get Application URL') {
            steps {
                script {
                    sh """
                        echo "=== Waiting for LoadBalancer IP ==="
                        kubectl get svc bankapp-service -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'
                    """
                }
            }
        }
    }
    
    post {
        success {
            echo 'Deployment successful!'
        }
        failure {
            echo 'Deployment failed!'
        }
    }
}
```

---

## 🚀 Deployment Process

### Manual Deployment

```bash
# 1. Clone the repository
git clone https://github.com/YOUR_USERNAME/BankApp-CD.git
cd BankApp-CD

# 2. Deploy MySQL Database
kubectl apply -f database/database-ds.yml

# 3. Wait for MySQL to be ready
kubectl rollout status deployment/mysql

# 4. Deploy BankApp
kubectl apply -f bankapp/bankapp-ds.yml

# 5. Wait for BankApp to be ready
kubectl rollout status deployment/bankapp

# 6. Get LoadBalancer URL
kubectl get svc bankapp-service
```

### Automated Deployment via Jenkins

1. Go to Jenkins Dashboard
2. Click on "CD" pipeline
3. Click "Build Now"
4. Monitor the build progress
5. Once complete, get the LoadBalancer URL from the logs

---

## 🔍 Verification Commands

### Check Deployment Status
```bash
# View all resources
kubectl get all

# Check pods
kubectl get pods -l app=bankapp
kubectl get pods -l app=mysql

# Check services
kubectl get svc

# Check deployments
kubectl get deployment
```

### View Logs
```bash
# BankApp logs
kubectl logs -l app=bankapp --tail=50

# MySQL logs
kubectl logs -l app=mysql --tail=50

# Follow logs in real-time
kubectl logs -f -l app=bankapp
```

### Describe Resources
```bash
# Describe pod
kubectl describe pod <POD_NAME>

# Describe service
kubectl describe svc bankapp-service

# Describe deployment
kubectl describe deployment bankapp
```

---

## 🔄 Update Deployment

### Update Docker Image
```bash
# Edit the manifest
vim bankapp/bankapp-ds.yml
# Change image tag: YOUR_USERNAME/bankapp:v25

# Apply changes
kubectl apply -f bankapp/bankapp-ds.yml

# Watch rollout
kubectl rollout status deployment/bankapp
```

### Rollback Deployment
```bash
# View rollout history
kubectl rollout history deployment/bankapp

# Rollback to previous version
kubectl rollout undo deployment/bankapp

# Rollback to specific revision
kubectl rollout undo deployment/bankapp --to-revision=2
```

---

## 🐛 Troubleshooting

### Pods Not Starting
```bash
# Check pod status
kubectl get pods

# Describe pod for events
kubectl describe pod <POD_NAME>

# Check logs
kubectl logs <POD_NAME>
```

### Image Pull Errors
```bash
# Verify image exists in Docker Hub
docker pull YOUR_USERNAME/bankapp:v24

# Check image name in manifest
kubectl get deployment bankapp -o yaml | grep image
```

### LoadBalancer Pending
```bash
# Check service
kubectl get svc bankapp-service

# Describe service for events
kubectl describe svc bankapp-service

# Wait 2-3 minutes for AWS to provision LoadBalancer
```

### Database Connection Issues
```bash
# Check MySQL pod
kubectl get pods -l app=mysql

# Check MySQL logs
kubectl logs -l app=mysql

# Verify service
kubectl get svc mysql-service

# Test connection from BankApp pod
kubectl exec -it <BANKAPP_POD> -- curl mysql-service:3306
```

---

## 📊 Monitoring

### Resource Usage
```bash
# Node resources
kubectl top nodes

# Pod resources
kubectl top pods

# Deployment status
kubectl get deployment -o wide
```

### Events
```bash
# View cluster events
kubectl get events --sort-by=.metadata.creationTimestamp

# Watch events in real-time
kubectl get events --watch
```

---

## 🔒 Security Best Practices

1. **Use Secrets for Sensitive Data**:
```bash
# Create secret for database password
kubectl create secret generic mysql-secret \
  --from-literal=password=Test@123

# Reference in deployment
env:
- name: MYSQL_ROOT_PASSWORD
  valueFrom:
    secretKeyRef:
      name: mysql-secret
      key: password
```

2. **Use ConfigMaps for Configuration**:
```bash
# Create ConfigMap
kubectl create configmap app-config \
  --from-literal=database.url=jdbc:mysql://mysql-service:3306/bankappdb

# Reference in deployment
envFrom:
- configMapRef:
    name: app-config
```

3. **Implement Resource Limits**:
```yaml
resources:
  requests:
    memory: "512Mi"
    cpu: "500m"
  limits:
    memory: "1Gi"
    cpu: "1000m"
```

---

## 📈 Scaling

### Manual Scaling
```bash
# Scale up
kubectl scale deployment bankapp --replicas=5

# Scale down
kubectl scale deployment bankapp --replicas=2
```

### Horizontal Pod Autoscaler
```bash
# Create HPA
kubectl autoscale deployment bankapp \
  --cpu-percent=70 \
  --min=2 \
  --max=10

# Check HPA status
kubectl get hpa
```

---

## 🧹 Cleanup

```bash
# Delete application
kubectl delete -f bankapp/bankapp-ds.yml

# Delete database
kubectl delete -f database/database-ds.yml

# Verify deletion
kubectl get all
```

---

## 📝 Notes

- LoadBalancer provisioning takes 2-3 minutes
- Database must be deployed before application
- Use `kubectl rollout status` to wait for deployments
- Check logs if pods are not starting
- Ensure AWS credentials are properly configured

---

## 🔗 Related Links

- [Kubernetes Documentation](https://kubernetes.io/docs/)
- [AWS EKS Documentation](https://docs.aws.amazon.com/eks/)
- [Jenkins Pipeline Documentation](https://www.jenkins.io/doc/book/pipeline/)

---

## 👨‍💻 Author
Your Name  
DevOps Engineer
