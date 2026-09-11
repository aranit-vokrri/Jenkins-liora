pipeline {
    agent any

    environment {
        IMAGE_NAME = 'aranit/lioraapi'
        TEST_CONTAINER = 'lioraapi-test'
    }

    stages {
        stage('Docker Build') {
            steps {
                sh '''
                    docker build \
                      -t ${IMAGE_NAME}:v.${BUILD_ID}.0 .
                '''
            }
        }

        stage('Docker Run') {
            steps {
                sh '''
                    docker rm -f ${TEST_CONTAINER} 2>/dev/null || true

                    docker run -d \
                      --name ${TEST_CONTAINER} \
                      -p 8000:8000 \
                      ${IMAGE_NAME}:v.${BUILD_ID}.0
                '''
            }
        }

        stage('Test Acceptance') {
            steps {
                sh '''
                    for attempt in 1 2 3 4 5; do
                        if curl --fail http://localhost:8000; then
                            exit 0
                        fi

                        sleep 2
                    done

                    docker logs ${TEST_CONTAINER}
                    exit 1
                '''
            }
        }

        stage('Docker Push') {
            steps {
                withCredentials([
                    string(
                        credentialsId: 'DOCKER_HUB_PASS',
                        variable: 'DOCKER_HUB_PASS'
                    )
                ]) {
                    sh '''
                        echo "$DOCKER_HUB_PASS" |
                          docker login \
                            --username aranit \
                            --password-stdin

                        docker push ${IMAGE_NAME}:v.${BUILD_ID}.0
                    '''
                }
            }
        }

        stage('Deployment in dev') {
            steps {
                withCredentials([
                    file(
                        credentialsId: 'config',
                        variable: 'KUBECONFIG_FILE'
                    )
                ]) {
                    sh '''
                        KUBECONFIG="$KUBECONFIG_FILE" \
                        helm upgrade --install app fastapi \
                          --namespace dev \
                          --set image.repository=${IMAGE_NAME} \
                          --set image.tag=v.${BUILD_ID}.0
                    '''
                }
            }
        }

        stage('Deployment in staging') {
            steps {
                withCredentials([
                    file(
                        credentialsId: 'config',
                        variable: 'KUBECONFIG_FILE'
                    )
                ]) {
                    sh '''
                        KUBECONFIG="$KUBECONFIG_FILE" \
                        helm upgrade --install app fastapi \
                          --namespace staging \
                          --set image.repository=${IMAGE_NAME} \
                          --set image.tag=v.${BUILD_ID}.0
                    '''
                }
            }
        }

        stage('Deployment in prod') {
            when {
                branch 'master'
            }

            steps {
                timeout(time: 15, unit: 'MINUTES') {
                    input(
                        message: 'Deploy this version to production?',
                        ok: 'Deploy'
                    )
                }

                withCredentials([
                    file(
                        credentialsId: 'config',
                        variable: 'KUBECONFIG_FILE'
                    )
                ]) {
                    sh '''
                        KUBECONFIG="$KUBECONFIG_FILE" \
                        helm upgrade --install app fastapi \
                          --namespace prod \
                          --set image.repository=${IMAGE_NAME} \
                          --set image.tag=v.${BUILD_ID}.0
                    '''
                }
            }
        }
    }

    post {
        always {
            sh '''
                docker rm -f ${TEST_CONTAINER} 2>/dev/null || true
                docker logout 2>/dev/null || true
            '''
        }
    }
}
