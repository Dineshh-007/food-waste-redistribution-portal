
pipeline {
    agent any

    parameters {
        booleanParam(
            name: 'TEST_FAULTY_RELEASE',
            defaultValue: false,
            description: 'Test faulty deployment and automatic rollback'
        )
    }

    environment {
        PATH = "/usr/local/bin:/opt/homebrew/bin:/usr/bin:/bin:/usr/sbin:/sbin"
        IMAGE_NAME = 'dineshh007/food-waste-portal'
        DEPLOYMENT = 'food-portal'
        CONTAINER_NAME = 'food-portal'
    }

    stages {
        stage('Checkout and Validate') {
            steps {
                sh '''
                    test -f index.html
                    test -f Dockerfile
                    test -f deployment.yaml
                    echo "Project files validated"
                '''
            }
        }

        stage('Check Tools') {
            steps {
                sh '''
                    docker --version
                    kubectl version --client
                    kubectl get nodes
                '''
            }
        }

        stage('Build Docker Image') {
            steps {
                sh 'docker build -t $IMAGE_NAME:v1 .'
            }
        }

        stage('Push Stable Image') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub-creds',
                        usernameVariable: 'DH_USER',
                        passwordVariable: 'DH_TOKEN'
                    )
                ]) {
                    sh '''
                        set +x
                        echo "$DH_TOKEN" | docker login \
                            -u "$DH_USER" --password-stdin
                        docker push $IMAGE_NAME:v1
                        docker logout
                    '''
                }
            }
        }

        stage('Deploy Stable Version') {
            steps {
                sh '''
                    kubectl apply -f deployment.yaml

                    kubectl set image deployment/$DEPLOYMENT \
                        $CONTAINER_NAME=$IMAGE_NAME:v1

                    kubectl rollout status deployment/$DEPLOYMENT \
                        --timeout=120s

                    kubectl get pods -l app=food-portal
                '''
            }
        }

        stage('Test Faulty Release and Roll Back') {
            when {
                expression { params.TEST_FAULTY_RELEASE }
            }
            steps {
                sh '''
                    echo "Deploying deliberately invalid image..."

                    kubectl set image deployment/$DEPLOYMENT \
                        $CONTAINER_NAME=$IMAGE_NAME:bad-tag

                    if kubectl rollout status deployment/$DEPLOYMENT \
                        --timeout=45s; then
                        echo "Faulty release unexpectedly succeeded"
                        exit 1
                    else
                        echo "Fault detected. Rolling back to stable version."

                        kubectl rollout undo deployment/$DEPLOYMENT

                        kubectl rollout status deployment/$DEPLOYMENT \
                            --timeout=120s

                        kubectl get deployment $DEPLOYMENT
                        kubectl get pods -l app=food-portal

                        echo "Rollback verified"
                    fi
                '''
            }
        }
    }

    post {
        success {
            echo 'Pipeline completed successfully.'
        }
        failure {
            echo 'Pipeline failed. Check the stage logs.'
        }
        always {
            echo 'Pipeline execution finished.'
        }
    }
}

