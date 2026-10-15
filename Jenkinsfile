
pipeline {
    agent any

    parameters {
        choice(
            name: 'DEPLOY_MODE',
            choices: ['NORMAL', 'FAULTY_TEST'],
            description: 'NORMAL deploys the portal. FAULTY_TEST verifies automatic rollback.'
        )
    }

    environment {
        DOCKER_IMAGE = 'abhijith006/student-placement-portal'
        DEPLOYMENT = 'placement-portal'
        CONTAINER_NAME = 'placement-portal'
        SERVICE_NAME = 'placement-portal-service'
    }

    stages {
        stage('Checkout') {
            steps {
                echo 'Source code checked out from GitHub.'
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    if (params.DEPLOY_MODE == 'FAULTY_TEST') {
                        bat 'docker build -f Dockerfile.faulty -t %DOCKER_IMAGE%:faulty-%BUILD_NUMBER% .'
                    } else {
                        bat 'docker build -t %DOCKER_IMAGE%:%BUILD_NUMBER% .'
                    }
                }
            }
        }

        stage('Push Image to Docker Hub') {
            steps {
                script {
                    def imageTag = params.DEPLOY_MODE == 'FAULTY_TEST'
                        ? "faulty-${env.BUILD_NUMBER}"
                        : env.BUILD_NUMBER

                    withCredentials([
                        string(
                            credentialsId: 'dockerhub-token',
                            variable: 'DOCKER_TOKEN'
                        )
                    ]) {
                        bat '''
                            powershell -NoProfile -Command "$env:DOCKER_TOKEN | docker login -u 'abhijith006' --password-stdin"
                        '''
                        bat "docker push %DOCKER_IMAGE%:${imageTag}"
                        bat 'docker logout'
                    }
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                script {
                    withCredentials([
                        file(
                            credentialsId: 'kubeconfig',
                            variable: 'KUBECONFIG'
                        )
                    ]) {
                        def imageTag = params.DEPLOY_MODE == 'FAULTY_TEST'
                            ? "faulty-${env.BUILD_NUMBER}"
                            : env.BUILD_NUMBER

                        // Apply the Service only; preserve the current Deployment.
                        bat 'kubectl apply -f deployment.yaml --dry-run=client -o yaml > nul'

                        bat '''
                            kubectl get service %SERVICE_NAME% > nul 2>&1
                        '''

                        // Deploy the selected image.
                        bat "kubectl set image deployment/%DEPLOYMENT% %CONTAINER_NAME%=%DOCKER_IMAGE%:${imageTag}"

                        int rolloutResult = bat(
                            returnStatus: true,
                            script: 'kubectl rollout status deployment/%DEPLOYMENT% --timeout=90s'
                        )

                        if (rolloutResult != 0) {
                            echo 'Deployment rollout failed. Attempting automatic rollback.'

                            bat 'kubectl rollout undo deployment/%DEPLOYMENT%'
                            bat 'kubectl rollout status deployment/%DEPLOYMENT% --timeout=90s'

                            error('New version failed. Previous Deployment revision restored.')
                        }
                    }
                }
            }
        }

        stage('Application Health Check') {
            steps {
                script {
                    withCredentials([
                        file(
                            credentialsId: 'kubeconfig',
                            variable: 'KUBECONFIG'
                        )
                    ]) {
                        int healthResult = bat(
                            returnStatus: true,
                            script: '''
                                kubectl run placement-health-%BUILD_NUMBER% --rm -i --restart=Never --image=curlimages/curl:8.12.1 --command -- curl -fsS --max-time 10 http://placement-portal-service.default.svc.cluster.local/
                            '''
                        )

                        if (healthResult != 0) {
                            echo 'Application health check failed. Rolling back.'

                            bat 'kubectl rollout undo deployment/%DEPLOYMENT%'
                            bat 'kubectl rollout status deployment/%DEPLOYMENT% --timeout=90s'

                            error('Health check failed. Rollback attempted.')
                        }
                    }
                }
            }
        }

        stage('Verify Deployment') {
            steps {
                withCredentials([
                    file(
                        credentialsId: 'kubeconfig',
                        variable: 'KUBECONFIG'
                    )
                ]) {
                    bat 'kubectl get deployment placement-portal'
                    bat 'kubectl get pods -o wide'
                    bat 'kubectl get service placement-portal-service'
                    bat 'kubectl get deployment placement-portal -o jsonpath="{.spec.template.spec.containers[0].image}"'
                    bat 'kubectl rollout history deployment/placement-portal'
                }
            }
        }
    }

    post {
        success {
            echo 'Pipeline completed successfully.'
        }
        failure {
            echo 'Pipeline failed. Check the console output and rollback status.'
        }
    }
}
