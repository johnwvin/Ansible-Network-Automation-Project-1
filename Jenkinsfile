pipeline {
    agent any

    environment {
        REGISTRY_URL     = "nexus.johnwvin.com"
        IMAGE_NAME       = "docker-hosted/ansible"
        IMAGE_TAG        = "3.13"
        IMAGE_FULL       = "${REGISTRY_URL}/${IMAGE_NAME}:${IMAGE_TAG}"
        PIP_INDEX_URL    = "https://nexus.johnwvin.com/repository/PyPi/simple"
        PIP_TRUSTED_HOST = "nexus.johnwvin.com"
        APT_MIRROR   = "https://nexus.johnwvin.com/repository/apt-deb/"

    }

    stages {
        stage('Docker Login') {
            steps {
                echo "🔐 Logging into Nexus Docker repository..."
                withCredentials([usernamePassword(
                    credentialsId: 'nexus-creds-1',
                    usernameVariable: 'NEXUS_USER',
                    passwordVariable: 'NEXUS_PASS'
                )]) {
                    sh '''
                        echo "$NEXUS_PASS" | docker login $REGISTRY_URL -u "$NEXUS_USER" --password-stdin
                    '''
                }
            }
        }

        stage('Check or Build Image') {
            steps {
                script {
                    echo "🔍 Checking if ${IMAGE_FULL} exists in Nexus..."
                    def exists = sh(
                        script: "docker pull ${IMAGE_FULL} >/dev/null 2>&1 && echo true || echo false",
                        returnStdout: true
                    ).trim()

                    if (exists == "true") {
                        echo "✅ Found cached image in Nexus, skipping build."
                    } else {
                        echo "⚙️ Image not found — building and pushing custom Ansible image..."

                        writeFile file: 'Dockerfile', text: '''
                            FROM python:3.13-slim

                            LABEL maintainer="johnwvin.com"
                            LABEL description="Custom Ansible + Ansible-Lint image (Python 3.13)"

                            RUN apt-get update && \
                                apt-get install -y --no-install-recommends git ssh curl ca-certificates && \
                                rm -rf /var/lib/apt/lists/*

                            # Preconfigure pip to use your Nexus PyPI mirror
                            ENV PIP_INDEX_URL=https://nexus.johnwvin.com/repository/PyPi/simple
                            ENV PIP_TRUSTED_HOST=nexus.johnwvin.com

                            RUN pip install --no-cache-dir --upgrade pip && \
                                pip install --no-cache-dir ansible ansible-lint

                            WORKDIR /workspace
                            CMD ["bash"]
                            '''

                        sh """
                            docker build -t ${IMAGE_FULL} .
                            docker push ${IMAGE_FULL}
                        """
                    }
                }
            }
        }

        stage('Run Lint in Container') {
            steps {
                script {
                    docker.image("${IMAGE_FULL}").inside('-u root:root') {
                        sh '''
                            echo "=== Using Nexus PyPI mirror: $PIP_INDEX_URL ==="
                            echo "=== Running ansible-lint ==="
                            ansible-lint -v playbooks/
                        '''
                    }
                }
            }
        }
    }

    post {
        always {
            echo "Pipeline completed (success or failure)."
        }
        success {
            echo "✅ Ansible lint check passed!"
        }
        failure {
            echo "❌ Ansible lint check failed!"
        }
    }
}
