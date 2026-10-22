
pipeline {
    agent any

    parameters {
        booleanParam(
            name: 'TEST_FAULTY_RELEASE',
            defaultValue: false,
            description: 'Introduce a faulty image and test automatic rollback'
        )
    }

    environment {
        IMAGE_NAME = 'dineshh007/food-waste-portal'
        DEPLOYMENT = 'food-portal'
        CONTAINER_NAME = 'food-portal'
    }

    stages {
        stage('Checkout and Validate') {
            steps {
                checkout scm
                sh '''
                    test -f index.html
                    test -f Dockerfile
                    test -f deployment.yaml
                    grep -qi "Food Waste Tracking" index.html
                    echo "Project files validated"
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
                    echo "Introducing an invalid image tag..."

                    kubectl set image deployment/$DEPLOYMENT \
                      $CONTAINER_NAME=$IMAGE_NAME:bad-tag

                    if kubectl rollout status deployment/$DEPLOYMENT \
                      --timeout=60s; then
                        echo "Unexpected: faulty rollout succeeded"
                        exit 1
                    else
                        echo "Fault detected. Rolling back..."
                        kubectl rollout undo deployment/$DEPLOYMENT
                        kubectl rollout status deployment/$DEPLOYMENT \
                          --timeout=120s
                    fi

                    kubectl get deployment $DEPLOYMENT
                    kubectl get pods -l app=food-portal
                    echo "Rollback verification complete"
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
