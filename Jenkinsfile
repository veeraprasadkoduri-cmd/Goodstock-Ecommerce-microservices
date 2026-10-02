```groovy
pipeline {
    agent any

    options {
        timestamps()
        disableConcurrentBuilds()
        buildDiscarder(logRotator(numToKeepStr: '10'))
        skipDefaultCheckout(true)
    }

    environment {
        AWS_REGION     = 'ap-south-1'
        AWS_ACCOUNT_ID = '426714791148'
        ECR_REGISTRY   = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"

        KUBECONFIG     = '/var/lib/jenkins/.kube/config'
        K8S_NAMESPACE  = 'devops-ecommerce'

        ROLLBACK_ENABLED = 'true'
    }

    stages {

        stage('Checkout') {
            steps {
                git(
                    branch: 'main',
                    url: 'https://github.com/veeraprasadkoduri-cmd/Goodstock-Ecommerce-microservices.git'
                )
            }
        }

        stage('Initialize') {
            steps {
                script {

                    def shortCommit = sh(
                        script: 'git rev-parse --short=7 HEAD',
                        returnStdout: true
                    ).trim()

                    env.IMAGE_TAG = "${env.BUILD_NUMBER}-${shortCommit}"

                    env.API_GATEWAY_IMAGE =
                        "${ECR_REGISTRY}/goodstock-api-gateway"

                    env.FRONTEND_IMAGE =
                        "${ECR_REGISTRY}/goodstock-frontend"

                    env.PRODUCT_IMAGE =
                        "${ECR_REGISTRY}/goodstock-product"

                    env.ORDER_IMAGE =
                        "${ECR_REGISTRY}/goodstock-order"

                    env.USER_IMAGE =
                        "${ECR_REGISTRY}/goodstock-user"

                    echo "========================================"
                    echo "AWS Region    : ${AWS_REGION}"
                    echo "ECR Registry  : ${ECR_REGISTRY}"
                    echo "Image Tag     : ${IMAGE_TAG}"
                    echo "========================================"
                }
            }
        }

        stage('Detect Changes') {
            steps {
                script {

                    def changedFiles = []

                    for (changeSet in currentBuild.changeSets) {
                        for (entry in changeSet.items) {
                            for (file in entry.affectedFiles) {
                                changedFiles.add(file.path)
                            }
                        }
                    }

                    changedFiles = changedFiles.unique().sort()

                    if (changedFiles.isEmpty()) {

                        echo "Jenkins changelog is empty."
                        echo "Checking current Git commit."

                        def fallbackOutput = sh(
                            script: '''
                                git diff-tree \
                                  --no-commit-id \
                                  --name-only \
                                  -r HEAD
                            ''',
                            returnStdout: true
                        ).trim()

                        if (fallbackOutput) {
                            changedFiles = fallbackOutput
                                .split('\\n')
                                .collect { it.trim() }
                                .findAll { it }
                                .unique()
                                .sort()
                        }
                    }

                    if (changedFiles.isEmpty()) {

                        echo "No reliable changed-file list available."
                        echo "Building all application services."

                        env.API_GATEWAY_CHANGED = 'true'
                        env.FRONTEND_CHANGED    = 'true'
                        env.PRODUCT_CHANGED     = 'true'
                        env.ORDER_CHANGED       = 'true'
                        env.USER_CHANGED        = 'true'

                        writeFile(
                            file: 'changed-files.txt',
                            text: 'Full application build\n'
                        )

                    } else {

                        writeFile(
                            file: 'changed-files.txt',
                            text: changedFiles.join('\n') + '\n'
                        )

                        echo "Changed files:"

                        changedFiles.each {
                            echo "  ${it}"
                        }

                        env.API_GATEWAY_CHANGED =
                            changedFiles.any {
                                it.startsWith('api-gateway/')
                            } ? 'true' : 'false'

                        env.FRONTEND_CHANGED =
                            changedFiles.any {
                                it.startsWith('frontend-gateway/')
                            } ? 'true' : 'false'

                        env.PRODUCT_CHANGED =
                            changedFiles.any {
                                it.startsWith('product-service/')
                            } ? 'true' : 'false'

                        env.ORDER_CHANGED =
                            changedFiles.any {
                                it.startsWith('order-service/')
                            } ? 'true' : 'false'

                        env.USER_CHANGED =
                            changedFiles.any {
                                it.startsWith('user-service/')
                            } ? 'true' : 'false'
                    }

                    env.ANY_APP_CHANGED = (
                        env.API_GATEWAY_CHANGED == 'true' ||
                        env.FRONTEND_CHANGED == 'true' ||
                        env.PRODUCT_CHANGED == 'true' ||
                        env.ORDER_CHANGED == 'true' ||
                        env.USER_CHANGED == 'true'
                    ) ? 'true' : 'false'

                    echo """
                    ========================================
                    CHANGE DETECTION
                    ========================================
                    API Gateway : ${env.API_GATEWAY_CHANGED}
                    Frontend    : ${env.FRONTEND_CHANGED}
                    Product     : ${env.PRODUCT_CHANGED}
                    Order       : ${env.ORDER_CHANGED}
                    User        : ${env.USER_CHANGED}
                    ----------------------------------------
                    Any App     : ${env.ANY_APP_CHANGED}
                    ========================================
                    """
                }
            }
        }

        stage('Validate') {
            steps {
                sh '''
                    set -eux

                    echo "Checking Git..."
                    git diff --check

                    echo "Checking Dockerfiles..."
                    test -f api-gateway/Dockerfile
                    test -f frontend-gateway/Dockerfile
                    test -f product-service/Dockerfile
                    test -f order-service/Dockerfile
                    test -f user-service/Dockerfile

                    echo "Checking Docker..."
                    docker --version

                    echo "Checking AWS CLI..."
                    aws --version

                    echo "Checking kubectl..."
                    kubectl version --client

                    echo "Checking AWS identity..."
                    aws sts get-caller-identity

                    echo "Checking Kubernetes..."
                    kubectl get nodes

                    echo "Checking namespace..."
                    kubectl get namespace ${K8S_NAMESPACE}
                '''
            }
        }

        stage('Build API Gateway') {
            when {
                expression {
                    env.API_GATEWAY_CHANGED == 'true'
                }
            }

            steps {
                sh '''
                    docker build \
                      -t ${API_GATEWAY_IMAGE}:${IMAGE_TAG} \
                      ./api-gateway
                '''
            }
        }

        stage('Build Frontend') {
            when {
                expression {
                    env.FRONTEND_CHANGED == 'true'
                }
            }

            steps {
                sh '''
                    docker build \
                      -t ${FRONTEND_IMAGE}:${IMAGE_TAG} \
                      ./frontend-gateway
                '''
            }
        }

        stage('Build Product Service') {
            when {
                expression {
                    env.PRODUCT_CHANGED == 'true'
                }
            }

            steps {
                sh '''
                    docker build \
                      -t ${PRODUCT_IMAGE}:${IMAGE_TAG} \
                      ./product-service
                '''
            }
        }

        stage('Build Order Service') {
            when {
                expression {
                    env.ORDER_CHANGED == 'true'
                }
            }

            steps {
                sh '''
                    docker build \
                      -t ${ORDER_IMAGE}:${IMAGE_TAG} \
                      ./order-service
                '''
            }
        }

        stage('Build User Service') {
            when {
                expression {
                    env.USER_CHANGED == 'true'
                }
            }

            steps {
                sh '''
                    docker build \
                      -t ${USER_IMAGE}:${IMAGE_TAG} \
                      ./user-service
                '''
            }
        }

        stage('Login to ECR') {
            when {
                expression {
                    env.ANY_APP_CHANGED == 'true'
                }
            }

            steps {
                sh '''
                    set -e

                    echo "Logging into Amazon ECR..."

                    aws ecr get-login-password \
                      --region ${AWS_REGION} | \
                    docker login \
                      --username AWS \
                      --password-stdin ${ECR_REGISTRY}
                '''
            }
        }

        stage('Push Images to ECR') {
            when {
                expression {
                    env.ANY_APP_CHANGED == 'true'
                }
            }

            steps {
                script {

                    if (env.API_GATEWAY_CHANGED == 'true') {
                        sh '''
                            docker push \
                              ${API_GATEWAY_IMAGE}:${IMAGE_TAG}
                        '''
                    }

                    if (env.FRONTEND_CHANGED == 'true') {
                        sh '''
                            docker push \
                              ${FRONTEND_IMAGE}:${IMAGE_TAG}
                        '''
                    }

                    if (env.PRODUCT_CHANGED == 'true') {
                        sh '''
                            docker push \
                              ${PRODUCT_IMAGE}:${IMAGE_TAG}
                        '''
                    }

                    if (env.ORDER_CHANGED == 'true') {
                        sh '''
                            docker push \
                              ${ORDER_IMAGE}:${IMAGE_TAG}
                        '''
                    }

                    if (env.USER_CHANGED == 'true') {
                        sh '''
                            docker push \
                              ${USER_IMAGE}:${IMAGE_TAG}
                        '''
                    }
                }
            }
        }

        stage('Deploy to K3s') {
            when {
                expression {
                    env.ANY_APP_CHANGED == 'true'
                }
            }

            steps {
                script {

                    echo "Deploying changed services to K3s..."

                    if (env.API_GATEWAY_CHANGED == 'true') {
                        sh '''
                            kubectl set image \
                              deployment/api-gateway \
                              api-gateway=${API_GATEWAY_IMAGE}:${IMAGE_TAG} \
                              -n ${K8S_NAMESPACE}
                        '''
                    }

                    if (env.FRONTEND_CHANGED == 'true') {
                        sh '''
                            kubectl set image \
                              deployment/frontend-gateway \
                              frontend-gateway=${FRONTEND_IMAGE}:${IMAGE_TAG} \
                              -n ${K8S_NAMESPACE}
                        '''
                    }

                    if (env.PRODUCT_CHANGED == 'true') {
                        sh '''
                            kubectl set image \
                              deployment/product-service \
                              product-service=${PRODUCT_IMAGE}:${IMAGE_TAG} \
                              -n ${K8S_NAMESPACE}
                        '''
                    }

                    if (env.ORDER_CHANGED == 'true') {
                        sh '''
                            kubectl set image \
                              deployment/order-service \
                              order-service=${ORDER_IMAGE}:${IMAGE_TAG} \
                              -n ${K8S_NAMESPACE}
                        '''
                    }

                    if (env.USER_CHANGED == 'true') {
                        sh '''
                            kubectl set image \
                              deployment/user-service \
                              user-service=${USER_IMAGE}:${IMAGE_TAG} \
                              -n ${K8S_NAMESPACE}
                        '''
                    }
                }
            }
        }

        stage('Verify Rollouts') {
            when {
                expression {
                    env.ANY_APP_CHANGED == 'true'
                }
            }

            steps {
                script {

                    if (env.API_GATEWAY_CHANGED == 'true') {
                        sh '''
                            kubectl rollout status \
                              deployment/api-gateway \
                              -n ${K8S_NAMESPACE} \
                              --timeout=180s
                        '''
                    }

                    if (env.FRONTEND_CHANGED == 'true') {
                        sh '''
                            kubectl rollout status \
                              deployment/frontend-gateway \
                              -n ${K8S_NAMESPACE} \
                              --timeout=180s
                        '''
                    }

                    if (env.PRODUCT_CHANGED == 'true') {
                        sh '''
                            kubectl rollout status \
                              deployment/product-service \
                              -n ${K8S_NAMESPACE} \
                              --timeout=180s
                        '''
                    }

                    if (env.ORDER_CHANGED == 'true') {
                        sh '''
                            kubectl rollout status \
                              deployment/order-service \
                              -n ${K8S_NAMESPACE} \
                              --timeout=180s
                        '''
                    }

                    if (env.USER_CHANGED == 'true') {
                        sh '''
                            kubectl rollout status \
                              deployment/user-service \
                              -n ${K8S_NAMESPACE} \
                              --timeout=180s
                        '''
                    }
                }
            }
        }

        stage('Smoke Tests') {
            when {
                expression {
                    env.ANY_APP_CHANGED == 'true'
                }
            }

            steps {
                sh '''
                    set -eux

                    echo "Running smoke tests..."

                    curl --fail \
                      --retry 5 \
                      --retry-delay 3 \
                      --max-time 10 \
                      http://localhost:30080/health

                    curl --fail \
                      --retry 3 \
                      --retry-delay 2 \
                      --max-time 15 \
                      http://localhost:30080/api/products

                    curl --fail \
                      --retry 3 \
                      --retry-delay 2 \
                      --max-time 15 \
                      http://localhost:30080/api/orders
                '''
            }
        }

        stage('Deployment Summary') {
            steps {
                sh '''
                    echo ""
                    echo "========================================"
                    echo "GOODSTOCK DEPLOYMENT SUMMARY"
                    echo "========================================"

                    echo "Image Tag: ${IMAGE_TAG}"

                    echo ""
                    echo "Changed Services:"
                    echo "API Gateway : ${API_GATEWAY_CHANGED}"
                    echo "Frontend    : ${FRONTEND_CHANGED}"
                    echo "Product     : ${PRODUCT_CHANGED}"
                    echo "Order       : ${ORDER_CHANGED}"
                    echo "User        : ${USER_CHANGED}"

                    echo ""
                    echo "Kubernetes Deployments:"

                    kubectl get deployments \
                      -n ${K8S_NAMESPACE} \
                      -o custom-columns='NAME:.metadata.name,IMAGE:.spec.template.spec.containers[*].image'

                    echo ""
                    echo "Pods:"

                    kubectl get pods \
                      -n ${K8S_NAMESPACE} \
                      -o wide

                    echo ""
                    echo "Services:"

                    kubectl get svc \
                      -n ${K8S_NAMESPACE}

                    echo ""
                    echo "========================================"
                '''
            }
        }
    }

    post {

        success {
            echo '========================================'
            echo 'CI/CD PIPELINE COMPLETED SUCCESSFULLY'
            echo '========================================'
            echo "Image Tag: ${env.IMAGE_TAG}"
        }

        failure {

            echo '========================================'
            echo 'PIPELINE FAILED'
            echo 'Starting automatic rollback...'
            echo '========================================'

            script {

                if (env.ROLLBACK_ENABLED == 'true') {

                    if (env.API_GATEWAY_CHANGED == 'true') {
                        sh '''
                            kubectl rollout undo \
                              deployment/api-gateway \
                              -n ${K8S_NAMESPACE} || true
                        '''
                    }

                    if (env.FRONTEND_CHANGED == 'true') {
                        sh '''
                            kubectl rollout undo \
                              deployment/frontend-gateway \
                              -n ${K8S_NAMESPACE} || true
                        '''
                    }

                    if (env.PRODUCT_CHANGED == 'true') {
                        sh '''
                            kubectl rollout undo \
                              deployment/product-service \
                              -n ${K8S_NAMESPACE} || true
                        '''
                    }

                    if (env.ORDER_CHANGED == 'true') {
                        sh '''
                            kubectl rollout undo \
                              deployment/order-service \
                              -n ${K8S_NAMESPACE} || true
                        '''
                    }

                    if (env.USER_CHANGED == 'true') {
                        sh '''
                            kubectl rollout undo \
                              deployment/user-service \
                              -n ${K8S_NAMESPACE} || true
                        '''
                    }
                }
            }

            echo 'Collecting Kubernetes diagnostics...'

            sh '''
                kubectl get pods \
                  -n ${K8S_NAMESPACE} \
                  -o wide || true

                echo ""

                kubectl get events \
                  -n ${K8S_NAMESPACE} \
                  --sort-by=.lastTime
```
