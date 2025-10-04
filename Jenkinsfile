pipeline {
    agent any

    environment {
        REGISTRY_URL      = "nexus.johnwvin.com"
        HOSTED_REPO       = "docker-hosted"
        GROUP_REPO        = "docker-group"
        IMAGE_NAME        = "custom-ansible"
        IMAGE_TAG         = "3.13"
        IMAGE_FULL_PUSH   = "${REGISTRY_URL}/${HOSTED_REPO}/${IMAGE_NAME}:${IMAGE_TAG}"
        IMAGE_FULL_PULL   = "${REGISTRY_URL}/${HOSTED_REPO}/${IMAGE_NAME}:${IMAGE_TAG}"
        PIP_INDEX_URL     = "https://nexus.johnwvin.com/repository/PyPi/simple"
        PIP_TRUSTED_HOST  = "nexus.johnwvin.com"
        APT_MIRROR        = "https://nexus.johnwvin.com/repository/apt-deb/"
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
                    echo "🔍 Checking if ${IMAGE_FULL_PULL} exists in Nexus..."
                    def exists = sh(
                        script: "docker pull ${IMAGE_FULL_PULL} >/dev/null 2>&1 && echo true || echo false",
                        returnStdout: true
                    ).trim()

                    if (exists == "true") {
                        echo "✅ Found cached image in Nexus group, skipping build."
                    } else {
                        echo "⚙️ Image not found — building and pushing custom Ansible image..."

                        writeFile file: 'Dockerfile', text: '''
                            FROM python:3.13-slim

                            LABEL maintainer="johnwvin.com"
                            LABEL description="Custom Ansible + Ansible-Lint image (Python 3.13)"

                            ARG APT_MIRROR
                            RUN echo "deb [trusted=yes] \$APT_MIRROR trixie main" > /etc/apt/sources.list && \
                                apt-get update && \
                                apt-get install -y --no-install-recommends git ssh curl ca-certificates && \
                                rm -rf /var/lib/apt/lists/*

                            # Pass pip mirror as build ARG
                            ARG PIP_INDEX_URL
                            ARG PIP_TRUSTED_HOST
                            ENV PIP_INDEX_URL=\$PIP_INDEX_URL
                            ENV PIP_TRUSTED_HOST=\$PIP_TRUSTED_HOST

                            RUN pip install --no-cache-dir --upgrade pip && \
                                pip install --no-cache-dir ansible ansible-lint

                            WORKDIR /workspace
                            CMD ["bash"]
                        '''

                        sh """
                            docker build \
                                --build-arg APT_MIRROR=${APT_MIRROR} \
                                --build-arg PIP_INDEX_URL=${PIP_INDEX_URL} \
                                --build-arg PIP_TRUSTED_HOST=${PIP_TRUSTED_HOST} \
                                -t ${IMAGE_FULL_PUSH} .
                            docker push ${IMAGE_FULL_PUSH}
                        """
                    }
                }
            }
        }

        stage('Run Lint in Container') {
            steps {
                script {
                    docker.image("${IMAGE_FULL_PULL}").inside('-u root:root') {
                        sh '''
                            echo "=== Using Nexus PyPI mirror: $PIP_INDEX_URL ==="
                            echo "=== Running ansible-lint ==="
                            ansible-lint -v playbooks/ --format json > ansible-lint-report.json || true
                        '''
                    }
                    // Archive so anyone can download the XML
                    archiveArtifacts artifacts: 'ansible-lint-report.json', allowEmptyArchive: true
                }
            }
        }
        stage('Report Lint to Gitea') {
            steps {
                script {
                    // Read the JSON report
                    def reportText = readFile 'ansible-lint-report.json'
                    def report = new groovy.json.JsonSlurper().parseText(reportText)
                    
                    // Count total failures
                    def failures = report.size()  // each entry is a violation
                    
                    // Compose a comment
                    def comment = "Ansible Lint completed: ${failures} violation(s). See Jenkins build artifacts for full report."
                    
                    // Post comment to Gitea
                    sh """
                    curl -s -X POST \
                        -H "Content-Type: application/json" \
                        -H "Authorization: token ${GITEA_TOKEN}" \
                        -d '{ "body": "${comment}" }' \
                        https://gitea.johnwvin.com/api/v1/repos/ORG/REPO/issues/${CHANGE_ID}/comments
                    """
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
