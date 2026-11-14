pipeline {
    agent any
    parameters {
        string(name: 'IMAGE_NAME', defaultValue: 'restaurant', description: 'Nom de l’image Docker')
        string(name: 'IMAGE_TAG', defaultValue: 'latest', description: 'Tag de l’image')
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
                    cat > Dockerfile <<'EOF'
                    FROM nginx:1.27-alpine
                    COPY . /usr/share/nginx/html/
                    EXPOSE 80
                    EOF
                    printf '.git\\nDockerfile\\n' > .dockerignore
                    docker build -t ${IMAGE_NAME}:${IMAGE_TAG} .
                '''
            }
        }
        stage('Acceptance Test') {
            steps {
                sh '''
                    TEST=${IMAGE_NAME}-test
                    JENKINS_NET=$(docker inspect jenkins -f '{{range $k, $v := .NetworkSettings.Networks}}{{$k}}{{end}}')
                    docker rm -f ${TEST} 2>/dev/null || true
                    docker run -d --name ${TEST} --network ${JENKINS_NET} ${IMAGE_NAME}:${IMAGE_TAG}
                    
                    for i in $(seq 1 10); do curl -sf http://${TEST}/ >/dev/null && break; sleep 1; done
                    
                    curl -sf http://${TEST}/ | grep -qi "<title>Restaurant</title>" && echo "✅ Titre OK"
                    docker rm -f ${TEST}
                '''
            }
        }
    }
}
