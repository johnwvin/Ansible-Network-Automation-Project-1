pipeline {
    agent any

    environment {
        PIP_INDEX_URL = 'https://nexus.johnwvin.com/repository/PyPi/'
        PIP_TRUSTED_HOST = 'nexus.johnwvin.com'
    }

    stages {
#        stage('Docker Login') {
#            steps {
#                withCredentials([usernamePassword(credentialsId: 'nexus-creds', usernameVariable: 'NEXUS_USER', passwordVariable: 'NEXUS_PASS')]) {
#                    sh '''
#                        echo "$NEXUS_PASS" | docker login nexus.johnwvin.com -u "$NEXUS_USER" --password-stdin
#                    '''
#                }
#            }
#        }

        stage('Pull Ansible Image') {
            steps {
                sh '''
                    echo "Pulling Ansible image from Nexus..."
                    docker pull nexus.johnwvin.com/ansible/ansible

                '''
            }
        }

        stage('Run Lint in Container') {
            steps {
                script {
                    docker.image('nexus.johnwvin.com/ansible/ansible').inside('-u root:root') {
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
