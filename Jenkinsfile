pipeline {
    agent any

    environment {
        REGISTRY_HOST  = 'harbor.local'
        IMAGE_NAME     = 'oracle/dba-dashboard'
        IMAGE_TAG      = "build-${BUILD_NUMBER}"
        ORACLE_TNS     = '//192.168.1.188:1521/ORCL'
        ORACLE_PDB     = 'ORCLPDB'
        SONAR_HOST     = 'http://192.168.1.184:9000'
        SLACK_WEBHOOK  = credentials('slack-webhook-url')
        REGISTRY_CREDS = credentials('harbor-registry-creds')
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
                withCredentials([string(credentialsId: 'sonarqube-token', variable: 'SONAR_TOKEN')]) {
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
                    docker build -t ${REGISTRY_HOST}/${IMAGE_NAME}:${IMAGE_TAG} .
                    docker tag ${REGISTRY_HOST}/${IMAGE_NAME}:${IMAGE_TAG} ${REGISTRY_HOST}/${IMAGE_NAME}:latest
                """
            }
        }

        stage('Push to Container Registry') {
            steps {
                echo "4. Pushing build artifacts to local container registry..."
                sh """
                    echo "${REGISTRY_CREDS_PSW}" | docker login ${REGISTRY_HOST} -u "${REGISTRY_CREDS_USR}" --password-stdin
                    docker push ${REGISTRY_HOST}/${IMAGE_NAME}:${IMAGE_TAG}
                    docker push ${REGISTRY_HOST}/${IMAGE_NAME}:latest
                """
            }
        }

        stage('Update GitOps K8s Manifest') {
            steps {
                echo "5. Updating K8s deployment manifest image tag..."
                sh """
                    sed -i 's|image: ${REGISTRY_HOST}/${IMAGE_NAME}:.*|image: ${REGISTRY_HOST}/${IMAGE_NAME}:${IMAGE_TAG}|g' k8s/deployment.yaml
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
                    "text": "✅ *Oracle DBA Dashboard Pipeline Completed!*\\n*Target DB:* `${ORACLE_TNS}` (`${ORACLE_PDB}`)\\n*Build Image Tag:* `${IMAGE_TAG}`\\n*Modules Deployed:* User Mgmt, Tablespaces, Instance Lifecycle, Performance, RMAN Backups, Audit History\\n*GitOps Engine:* Argo CD auto-syncing to Minikube cluster."
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
    }
}
