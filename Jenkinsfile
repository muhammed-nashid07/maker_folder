
pipeline {
    agent any

    environment {
        DOCKER_CREDENTIALS_ID = 'docker-cred'
        DOCKER_IMAGE          = 'muhammednashid/maker'

        SONAR_SERVER_NAME     = 'sonarqube-env'
        SONAR_SCANNER_TOOL    = 'sonar-scanner'

        GIT_CREDENTIALS_ID    = 'github-cred'
        MANIFEST_REPO         = 'https://github.com/muhammed-nashid07/maker_folder.git'
        MANIFEST_BRANCH       = 'main'
    }

    stages {

        // Stage 1: Checkout Source Code
        stage('Checkout Source Code') {
            steps {
                echo "Source code checked out successfully"
            }
        }

        // Stage 2: SonarQube Analysis
        stage('SonarQube Analysis') {
            steps {
                script {
                    def scannerHome = tool "${SONAR_SCANNER_TOOL}"

                    withSonarQubeEnv("${SONAR_SERVER_NAME}") {
                        sh """
                            ${scannerHome}/bin/sonar-scanner \
                            -Dsonar.projectKey=maker-key \
                            -Dsonar.projectName=maker-project \
                            -Dsonar.sources=. \
                            -Dsonar.sourceEncoding=UTF-8
                        """
                    }
                }
            }
        }

        // Stage 3: Build and Push Docker Image
        stage('Build & Push Docker Image') {
            steps {
                script {
                    withCredentials([
                        usernamePassword(
                            credentialsId: "${DOCKER_CREDENTIALS_ID}",
                            usernameVariable: 'DOCKER_USER',
                            passwordVariable: 'DOCKER_PASS'
                        )
                    ]) {
                        sh """
                            echo "\$DOCKER_PASS" | docker login \
                            -u "\$DOCKER_USER" --password-stdin

                            docker build \
                            -t ${DOCKER_IMAGE}:${BUILD_NUMBER} \
                            -t ${DOCKER_IMAGE}:latest \
                            -f Dockerfile .

                            docker push ${DOCKER_IMAGE}:${BUILD_NUMBER}

                            docker push ${DOCKER_IMAGE}:latest

                            docker logout
                        """
                    }
                }
            }
        }

        // Stage 4: Update Kubernetes Manifest Repository
        stage('Update Manifest Repo') {
            steps {
                dir('manifest-workspace') {

                    git branch: "${MANIFEST_BRANCH}",
                        credentialsId: "${GIT_CREDENTIALS_ID}",
                        url: "${MANIFEST_REPO}"

                    sh """
                        git config user.name "muhammed-nashid07"
                        git config user.email "muhammadnashid25@gmail.com"

                        # Update Kubernetes deployment image
                        sed -i '/name: maker/{n;s|image:.*|image: ${DOCKER_IMAGE}:${BUILD_NUMBER}|;}' \
                        manifests/deployment.yaml

                        # Check for changes before committing
                        if ! git diff --quiet; then

                            git add manifests/deployment.yaml

                            git commit -m "chore(manifest): update maker image tag to ${BUILD_NUMBER} [skip ci]"

                            git push origin ${MANIFEST_BRANCH}

                        else
                            echo "No manifest changes detected."
                        fi
                    """
                }
            }
        }
    }

    post {
        success {
            echo 'Pipeline executed successfully! Docker image pushed and Kubernetes manifest updated.'
        }

        failure {
            echo 'Pipeline failed. Please review the stage logs above.'
        }

        always {
            echo 'Pipeline execution completed.'
        }
    }
}

