pipeline {
    agent any

    environment {
        DOCKER_IMAGE = 'abhijith006/student-placement-portal'
        DEPLOYMENT = 'placement-portal'
        CONTAINER_NAME = 'placement-portal'
    }

    stages {

        stage('Clone Code') {
            steps {
                echo 'Source code is checked out from GitHub.'
            }
        }

        stage('Build Docker Image') {
            steps {
                bat 'docker build -t %DOCKER_IMAGE%:%BUILD_NUMBER% .'
            }
        }

        stage('Push Image to Docker Hub') {
            steps {
                withCredentials([
                    string(
                        credentialsId: 'dockerhub-token',
                        variable: 'DOCKER_TOKEN'
                    )
                ]) {
                    bat '''
                        powershell -NoProfile -Command "$env:DOCKER_TOKEN | docker login -u 'abhijith006' --password-stdin"
                    '''

                    bat 'docker push %DOCKER_IMAGE%:%BUILD_NUMBER%'

                    bat 'docker logout'
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
                        bat 'kubectl apply -f deployment.yaml'

                        bat '''
                            kubectl set image deployment/%DEPLOYMENT% %CONTAINER_NAME%=%DOCKER_IMAGE%:%BUILD_NUMBER%
                        '''

                        int rolloutResult = bat(
                            returnStatus: true,
                            script: 'kubectl rollout status deployment/%DEPLOYMENT% --timeout=90s'
                        )

                        if (rolloutResult != 0) {
                            echo 'Deployment rollout failed. Rolling back.'
                            bat 'kubectl rollout undo deployment/%DEPLOYMENT%'
                            bat 'kubectl rollout status deployment/%DEPLOYMENT% --timeout=90s'

                            error('New version failed to deploy. Previous version restored.')
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
                                kubectl run placement-health-check --rm -i --restart=Never --image=curlimages/curl:8.12.1 --command -- curl -fsS --max-time 5 http://placement-portal-service.default.svc.cluster.local/
                            '''
                        )

                        if (healthResult != 0) {
                            echo 'Health check failed. Rolling back to the previous release.'

                            bat 'kubectl rollout undo deployment/%DEPLOYMENT%'
                            bat 'kubectl rollout status deployment/%DEPLOYMENT% --timeout=90s'

                            error('Application health check failed. Rollback completed.')
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
                    bat 'kubectl get deployments'
                    bat 'kubectl get pods -o wide'
                    bat 'kubectl get services'
                    bat 'kubectl rollout history deployment/placement-portal'
                }
            }
        }
    }
}
