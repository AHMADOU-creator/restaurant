pipeline {
    agent any
    parameters {
        string(name: 'IMAGE_NAME', defaultValue: 'restaurant', description: 'Nom de l’image Docker')
        string(name: 'IMAGE_TAG', defaultValue: '1.0', description: 'Tag de version')
        string(name: 'DOCKERHUB_ID', defaultValue: 'TON_ID_DOCKERHUB', description: 'Votre identifiant Docker Hub')
    }
    stages {
        stage('Git Checkout') {
            steps {
                checkout scm
            }
        }
        stage('Docker Build') {
            steps {
                sh '''
                    UNIQUE_TAG=${IMAGE_TAG}.${BUILD_NUMBER}-${GIT_COMMIT:0:7}
                    echo "${UNIQUE_TAG}" > tag.txt
                    
                    cat > Dockerfile <<'EOF'
                    FROM nginx:1.27-alpine
                    COPY . /usr/share/nginx/html/
                    EXPOSE 80
                    EOF
                    printf '.git\\nDockerfile\\n' > .dockerignore
                    
                    docker build --label org.opencontainers.image.revision=${GIT_COMMIT} \
                                 -t ${DOCKERHUB_ID}/${IMAGE_NAME}:${UNIQUE_TAG} \
                                 -t ${IMAGE_NAME}:${IMAGE_TAG} .
                '''
            }
        }
        stage('Acceptance Test') {
            steps {
                sh '''
                    UNIQUE_TAG=$(cat tag.txt)
                    TEST=${IMAGE_NAME}-test
                    IMAGE=${DOCKERHUB_ID}/${IMAGE_NAME}
                    
                    JENKINS_NET=$(docker inspect jenkins -f '{{range $k, $v := .NetworkSettings.Networks}}{{$k}}{{end}}')
                    docker rm -f ${TEST} 2>/dev/null || true
                    docker run -d --name ${TEST} --network ${JENKINS_NET} ${IMAGE}:${UNIQUE_TAG}
                    
                    for i in $(seq 1 10); do curl -sf http://${TEST}/ >/dev/null && break; sleep 1; done
                    
                    curl -sf http://${TEST}/ | grep -qi "<title>Restaurant</title>" && echo "✅ Titre OK"
                    docker rm -f ${TEST}
                '''
            }
        }
        stage('Publish to Docker Hub') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub', passwordVariable: 'DOCKERHUB_TOKEN', usernameVariable: 'DOCKERHUB_USER')]) {
                    sh '''
                        UNIQUE_TAG=$(cat tag.txt)
                        IMAGE=${DOCKERHUB_ID}/${IMAGE_NAME}
                        
                        echo "${DOCKERHUB_TOKEN}" | docker login -u "${DOCKERHUB_USER}" --password-stdin
                        docker push ${IMAGE}:${UNIQUE_TAG}
                        docker tag ${IMAGE}:${UNIQUE_TAG} ${IMAGE}:latest
                        docker push ${IMAGE}:latest
                        docker logout
                        echo "Image publiée avec succès : ${IMAGE}:${UNIQUE_TAG}"
                    '''
                }
            }
        }
    }
    post {
        always {
            sh 'docker rm -f ${IMAGE_NAME}-test 2>/dev/null || true'
        }
    }
}
