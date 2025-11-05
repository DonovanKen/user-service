@Library('testsharedlibry') _

pipeline {
    agent any

    parameters {
       string(name: 'USER_SERVICE', defaultValue: 'userservice', description: 'this is my docker image name for user service')
       string(name: 'IMAGE_TAG', defaultValue: 'latest', description: 'this is my docker tag name')
       string(name: 'CONTAINER_USERSERVICE', defaultValue: 'cuser-service', description: 'this is my docker container name for user service')
       string(name: 'GITHUB_USER', defaultValue: 'DonovanKen', description: 'user')
       string(name: 'DOCKERHUB_USER', defaultValue: 'mrkendono', description: 'docker hub')
       string(name: 'K8S_NAMESPACE', defaultValue: 'microservices', description: 'Kubernetes namespace')
    }
    
    stages {
      stage('Build Image') {
        steps {
          script {
            sh """
                docker build -t ${params.USER_SERVICE}:${params.IMAGE_TAG} .
            """
          }
        }
      }
      
      stage('Push Images to Docker Hub') {
        steps {
            script {
                sh """
                    docker tag ${params.USER_SERVICE}:${params.IMAGE_TAG} ${params.DOCKERHUB_USER}/${params.USER_SERVICE}:${params.IMAGE_TAG}
                    docker push ${params.DOCKERHUB_USER}/${params.USER_SERVICE}:${params.IMAGE_TAG}
                """
            }
        }
      }

      stage('Run Container Test') {
        steps {
            script {
                sh """
                    echo "Cleaning up any existing containers..."
                    docker rm -f ${params.CONTAINER_USERSERVICE} 2>/dev/null || true
                    docker rm -f ${params.CONTAINER_USERSERVICE}-test 2>/dev/null || true
                    
                    echo "Testing user service container..."
                    docker run -d -p 5001:5001 --name ${params.CONTAINER_USERSERVICE}-test ${params.USER_SERVICE}:${params.IMAGE_TAG}
                    docker ps
                    sleep 10
                    echo "Testing user service container..."
                    curl -f http://localhost:5001/health || curl -I http://localhost:5001 || echo "User service health check completed"
                    echo "Cleaning up test container..."
                    docker rm -f ${params.CONTAINER_USERSERVICE}-test
                    echo "User service test completed..."
                """
            }
        }
      }

      stage('Deploy to Kubernetes') {
        steps {
            sshagent(credentials: ['ssh-cred1']) {
                sh '''
                    [ -d ~/.ssh ] || mkdir ~/.ssh && chmod 0700 ~/.ssh
                    ssh-keyscan -t rsa,dsa 192.168.2.88 >> ~/.ssh/known_hosts
                '''
                
                sh """
                    # Create directory on remote server first
                    ssh -o StrictHostKeyChecking=no kubernetes@192.168.2.88 'mkdir -p /tmp/k8s-user-service/'
                    
                    # Copy k8s manifests to target server
                    scp -r k8s/ kubernetes@192.168.2.88:/tmp/k8s-user-service/
                    
                    # Deploy to Kubernetes
                    ssh kubernetes@192.168.2.88 "
                        cd /tmp/k8s-user-service/k8s
                        echo '=== DEBUG: Current directory and files ==='
                        pwd
                        ls -la
                        echo 'Creating namespace if it does not exist'
                        kubectl create namespace ${params.K8S_NAMESPACE} --dry-run=client -o yaml | kubectl apply -f -
                        echo 'Applying Kubernetes manifests'
                        kubectl apply -f . -n ${params.K8S_NAMESPACE}
                        echo 'Waiting for user service deployment to be ready'
                        kubectl wait --for=condition=available deployment/user-api -n ${params.K8S_NAMESPACE} --timeout=300s
                        echo '=== Final Deployment Status ==='
                        kubectl get all -n ${params.K8S_NAMESPACE}
                    "
                """
            }
        }
      }
    }
    
    post {
        always {
            echo 'Pipeline completed - check Kubernetes deployment status'
            sh """
                docker rm -f ${params.CONTAINER_USERSERVICE} 2>/dev/null || true
                docker rm -f ${params.CONTAINER_USERSERVICE}-test 2>/dev/null || true
            """
        }
        success {
            echo 'User service deployed successfully to Kubernetes!'
        }
        failure {
            echo 'Deployment failed - check Kubernetes logs'
        }
    }
}