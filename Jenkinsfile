pipeline {
    agent {
        docker {
            // Use your Ansible image from your Nexus Docker registry repository URL, like the PyPI example
            image 'nexus.johnwvin.com/repository/docker/ansible:latest'
            args '-u root:root'
        }
    }

    environment {
        PIP_INDEX_URL = 'https://nexus.johnwvin.com/repository/PyPi/'
        PIP_TRUSTED_HOST = 'nexus.johnwvin.com'
    }

    stages {
        stage('Setup') {
            steps {
                sh '''
                    echo "=== Installing dependencies via Nexus PyPI ==="
                    pip install --no-cache-dir --upgrade pip
                    pip install --index-url $PIP_INDEX_URL ansible ansible-lint
                '''
            }
        }

        stage('Lint Playbooks') {
            steps {
                sh '''
                    echo "=== Running ansible-lint ==="
                    ansible-lint -v playbooks/
                '''
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