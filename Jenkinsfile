pipeline {
    agent any

    environment {
        DOCKERHUB_REPO = 'bus57790/oracle-dba-lab'
        IMAGE_TAG      = "build-${BUILD_NUMBER}"
        ORACLE_TNS     = '//192.168.1.188:1521/ORCL'
        SONAR_HOST     = 'http://192.168.1.184:9000'
        SLACK_WEBHOOK  = credentials('slack-webhook-url')
    }

    stages {
        stage('Checkout Code') {
            steps {
                echo "1. Pulling latest codebase for Oracle 19c DBA Dashboard..."
                git branch: 'main', url: 'https://github.com/bus57790/oracle-dba-lab.git'
            }
        }

        stage('SonarQube Code Quality Analysis') {
            steps {
                echo "2. Running static analysis on single-file React DBA Dashboard..."
                withCredentials([string(credentialsId: 'sonar-token', variable: 'SONAR_TOKEN')]) {
                    sh """
                        docker run --rm \
                        -v "${WORKSPACE}:/usr/src" \
                        sonarsource/sonar-scanner-cli \
                        -Dsonar.projectKey=oracle-dba-dashboard \
                        -Dsonar.sources=frontend \
                        -Dsonar.host.url=${SONAR_HOST} \
                        -Dsonar.login=${SONAR_TOKEN}
                    """
                }
            }
        }

        stage('Build & Tag Docker Container') {
            steps {
                echo "3. Packaging Nginx container image with interactive DBA App..."
                sh """
                    docker build -t ${DOCKERHUB_REPO}:${IMAGE_TAG} .
                    docker tag ${DOCKERHUB_REPO}:${IMAGE_TAG} ${DOCKERHUB_REPO}:latest
                """
            }
        }

        stage('Push to Docker Hub') {
            steps {
                echo "4. Pushing build artifacts to Docker Hub..."
                withCredentials([usernamePassword(credentialsId: 'dockerhub-creds', usernameVariable: 'DOCKERHUB_USER', passwordVariable: 'DOCKERHUB_PASS')]) {
                    sh """
                        echo "${DOCKERHUB_PASS}" | docker login -u "${DOCKERHUB_USER}" --password-stdin
                        docker push ${DOCKERHUB_REPO}:${IMAGE_TAG}
                        docker push ${DOCKERHUB_REPO}:latest
                        docker logout
                    """
                }
            }
        }

        stage('Update GitOps K8s Manifest') {
            steps {
                echo "5. Updating K8s deployment manifest image tag..."
                sh """
                    sed -i 's|image: ${DOCKERHUB_REPO}:.*|image: ${DOCKERHUB_REPO}:${IMAGE_TAG}|g' k8s/deployment.yaml
                    git config user.name "Jenkins CI Engine"
                    git config user.email "jenkins@home-lab.local"
                    git add k8s/deployment.yaml
                    git commit -m "ci(dba-dashboard): update image tag to ${IMAGE_TAG} [skip ci]" || echo "No changes to commit"
                    git push origin main
                """
            }
        }
    }

    post {
        success {
            sh """
                curl -X POST -H 'Content-type: application/json' \
                --data '{
                    "text": "✅ *Oracle DBA Dashboard Pipeline Completed!*\\n*Target DB:* `${ORACLE_TNS}`\\n*Build Image Tag:* `${IMAGE_TAG}`\\n*Modules Deployed:* User Mgmt, Tablespaces, Instance Lifecycle, Performance, RMAN Backups, Audit History\\n*GitOps Engine:* Argo CD auto-syncing to Minikube cluster."
                }' \
                ${SLACK_WEBHOOK}
            """
        }
        failure {
            sh """
                curl -X POST -H 'Content-type: application/json' \
                --data '{
                    "text": "❌ *Oracle DBA Dashboard Pipeline Failed!*\\n*Build Number:* #${BUILD_NUMBER}\\n*Target DB:* `${ORACLE_TNS}`\\nPlease inspect Jenkins build logs."
                }' \
                ${SLACK_WEBHOOK}
            """
        }
        always {
            sh "docker rmi ${DOCKERHUB_REPO}:${IMAGE_TAG} ${DOCKERHUB_REPO}:latest || true"
        }
    }
}
