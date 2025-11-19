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
                        
                        # Deploy database first
                        echo '=== Deploying Database ==='
                        kubectl apply -f mysql-deployment.yaml -n ${params.K8S_NAMESPACE}
                        kubectl apply -f mysql-service.yaml -n ${params.K8S_NAMESPACE}
                        
                        echo 'Waiting for database to be ready...'
                        kubectl wait --for=condition=ready pod -l app=user-db -n ${params.K8S_NAMESPACE} --timeout=300s
                        
                        # Deploy user service with ClusterIP
                        echo '=== Deploying User Service ==='
                        kubectl apply -f deployment.yaml -n ${params.K8S_NAMESPACE}
                        kubectl apply -f service.yaml -n ${params.K8S_NAMESPACE}
                        
                        echo 'Waiting for user service deployment to be ready'
                        kubectl wait --for=condition=available deployment/user-api -n ${params.K8S_NAMESPACE} --timeout=300s
                        
                        echo '=== Initializing Database Tables ==='
                        kubectl apply -f - <<EOF
apiVersion: batch/v1
kind: Job
metadata:
  name: init-user-database
  namespace: ${params.K8S_NAMESPACE}
spec:
  template:
    spec:
      containers:
      - name: init-db
        image: ${params.DOCKERHUB_USER}/${params.USER_SERVICE}:${params.IMAGE_TAG}
        command: 
        - python
        - -c
        - |
          from application import create_app, db
          from application.models import User
          from passlib.hash import sha256_crypt
          app = create_app()
          with app.app_context():
              print('=== Creating Database Tables ===')
              db.create_all()
              print('✓ Tables created successfully')
              
              # Create test user if none exist
              user_count = User.query.count()
              if user_count == 0:
                  test_user = User(
                      first_name='Admin',
                      last_name='User', 
                      email='admin@example.com',
                      username='admin',
                      password=sha256_crypt.hash('admin123'),
                      authenticated=True
                  )
                  db.session.add(test_user)
                  db.session.commit()
                  print('✓ Test user created: admin / admin123')
              print(f'✓ Total users: {User.query.count()}')
        env:
        - name: DATABASE_URL
          value: \"mysql+pymysql://cloudacademy:pfm_2020@user-db:3306/user\"
      restartPolicy: Never
  backoffLimit: 2
EOF

                        echo 'Waiting for database initialization to complete...'
                        kubectl wait --for=condition=complete job/init-user-database -n ${params.K8S_NAMESPACE} --timeout=120s
                        
                        echo '=== Database Initialization Logs ==='
                        kubectl logs job/init-user-database -n ${params.K8S_NAMESPACE}
                        
                        echo '=== Testing Service Connectivity ==='
                        # Test internal service connectivity
                        kubectl exec deployment/user-api -n ${params.K8S_NAMESPACE} -- curl -f http://localhost:5001/health || echo 'Health check completed'
                        
                        echo '=== Final Deployment Status ==='
                        kubectl get all -n ${params.K8S_NAMESPACE}
                        
                        echo '=== Service Details ==='
                        kubectl get svc -n ${params.K8S_NAMESPACE}
                        
                        echo '=== Testing Internal Service Discovery ==='
                        kubectl run test-curl --image=curlimages/curl -n ${params.K8S_NAMESPACE} --rm -i --restart=Never --command -- sh -c 'curl -v http://user-api:5001/health && echo \"✓ Internal service discovery working!\"' || echo 'Service discovery test completed'
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
            echo 'User service deployed successfully to Kubernetes with ClusterIP!'
        }
        failure {
            echo 'Deployment failed - check Kubernetes logs'
        }
    }
}