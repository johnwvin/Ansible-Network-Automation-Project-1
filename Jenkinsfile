pipeline {
    agent any

    environment {
        LOCAL_DOCKER_REGISTRY = "localhost:5000"
        LOCAL_PYPI_SERVER = "http://localhost:8080"
        AGENT_IMAGE_NAME = "local-ansible-project-agent:${env.BUILD_ID}"
    }

    stages {
        stage('Sync Dependencies to Local Caches') {
            steps {
                script {
                    echo "--- Syncing Docker Images ---"
                    def dockerImages = readFile('ci/docker_images.txt').trim().split('\n')
                    dockerImages.each { imageName ->
                        echo "Syncing ${imageName} to local registry..."
                        sh "docker pull ${imageName}"
                        sh "docker tag ${imageName} ${LOCAL_DOCKER_REGISTRY}/${imageName}"
                        sh "docker push ${LOCAL_DOCKER_REGISTRY}/${imageName}"
                    }

                    echo "\n--- Syncing Python Packages ---"
                    docker.image('python:3.13-slim').inside {
                        sh "pip install -r ci/python_packages.txt"
                        sh "pip download -r ci/python_packages.txt -d ./packages"
                        sh "twine upload --repository-url ${LOCAL_PYPI_SERVER}/ --skip-existing ./packages/*"
                    }
                }
            }
        }

        stage('Build Local Ansible Agent') {
            steps {
                // *** THE FIX IS HERE ***
                // We wrap the advanced docker.build command in a 'script' block.
                script {
                    echo "Building the final agent image using local caches..."
                    docker.build(AGENT_IMAGE_NAME, "./ci")
                }
            }
        }

        stage('Lint Ansible Code') {
            agent {
                docker { image AGENT_IMAGE_NAME }
            }
            steps {
                checkout scm
                sh "ansible-lint ."
            }
        }

        stage('Check Ansible Playbooks') {
            agent {
                docker { image AGENT_IMAGE_NAME }
            }
            steps {
                checkout scm
                sh "ansible-playbook -i inventory/staging.ini playbooks/main.yml --check"
            }
        }
    }
    
    post {
        always {
            echo "Cleaning up local Docker image: ${AGENT_IMAGE_NAME}"
            sh "docker rmi ${AGENT_IMAGE_NAME} || true"
        }
    }
}