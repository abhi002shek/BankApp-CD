pipeline {
    agent any
    
    environment {
        GIT_REPO = 'https://github.com/abhi002shek/BankApp-CD.git'
        GIT_BRANCH = 'main'
        EKS_CLUSTER = 'devopsshack-cluster'
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
                        kubectl get svc bankapp-service -o jsonpath='{.status.loadBalancer.ingress[0].hostname}' > lb_url.txt || echo "Pending..." > lb_url.txt
                        cat lb_url.txt
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
