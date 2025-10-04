pipeline {
    agent any

    environment {
        PIP_INDEX_URL = 'https://nexus.johnwvin.com/repository/PyPi/'
        PIP_TRUSTED_HOST = 'nexus.johnwvin.com'
    }

    stages {

        stage('Pull Ansible Image') {
            steps {
                sh '''
                    echo "Pulling Ansible image from Nexus..."
                    docker pull nexus.johnwvin.com:443/ansible

                '''
            }
        }

        stage('Run Lint in Container') {
            steps {
                script {
                    docker.image('nexus.johnwvin.com:443/ansible').inside('-u root:root') {
                        sh '''
                            echo "=== Installing dependencies via Nexus PyPI ==="
                            pip install --no-cache-dir --upgrade pip
                            pip install --index-url $PIP_INDEX_URL ansible ansible-lint
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
            echo 'Pipeline completed (success or failure).'
        }
        success {
            echo '✅ Ansible lint check passed!'
        }
        failure {
            echo '❌ Ansible lint check failed!'
        }
    }
}
