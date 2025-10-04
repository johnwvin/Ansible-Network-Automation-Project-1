pipeline {
    agent any

    environment {
        LOCAL_DOCKER_REGISTRY = "localhost:5000"
        LOCAL_PYPI_SERVER = "http://localhost:8080"
        AGENT_IMAGE_NAME = "local-ansible-project-agent:${env.BUILD_ID}"
    }

    stages {
        stage('Setup and Ensure Local Services') {
            steps {
                script {
                    sh '''
                        #!/bin/bash
                        SERVICE_DIR="/opt/jenkins-services"
                        COMPOSE_FILE="${SERVICE_DIR}/docker-compose.yml"
                        if [ ! -d "${SERVICE_DIR}" ]; then
                          sudo mkdir -p "${SERVICE_DIR}/docker-registry-data"
                          sudo mkdir "${SERVICE_DIR}/pypi-server-packages"
                        fi
                        if [ ! -f "${COMPOSE_FILE}" ]; then
                          sudo tee "${COMPOSE_FILE}" > /dev/null <<EOF
version: '3.7'
services:
  registry:
    image: registry:2
    restart: always
    ports:
      - "5000:5000"
    volumes:
      - ./docker-registry-data:/var/lib/registry
  pypi:
    image: pypiserver/pypiserver:latest
    restart: always
    ports:
      - "8080:8080"
    volumes:
      - ./pypi-server-packages:/data/packages
EOF
                        fi
                        cd ${SERVICE_DIR}
                        RUNNING_SERVICES=$(sudo docker-compose ps | grep "Up" | wc -l)
                        if [ "$RUNNING_SERVICES" -lt 2 ]; then
                          sudo docker-compose up -d
                          sleep 5
                        fi
                    '''
                }
            }
        }

        stage('Sync Dependencies to Local Caches') {
            steps {
                script {
                    echo "--- Syncing Docker Images ---"
                    def dockerImages = readFile('ci/docker_images.txt').trim().split('\n')
                    dockerImages.each { imageName ->
                        sh "docker pull ${imageName}"
                        sh "docker tag ${imageName} ${LOCAL_DOCKER_REGISTRY}/${imageName}"
                        sh "docker push ${LOCAL_DOCKER_REGISTRY}/${imageName}"
                    }

                    echo "\n--- Syncing Python Packages ---"
                    docker.image('python:3.13-slim').inside("-u root") {
                        sh "pip install -r ci/python_packages.txt"
                        sh "pip download -r ci/python_packages.txt -d ./packages"
                        sh '''
                            #!/bin/bash
                            for pkg in ./packages/*.whl; do
                              PKG_NAME=$(basename ${pkg} | cut -d- -f1)
                              echo "Checking for package: ${PKG_NAME} on local server..."
                              if pip search --index ${LOCAL_PYPI_SERVER}/simple ${PKG_NAME} | grep -i ${PKG_NAME}; then
                                echo "Package ${PKG_NAME} already exists. Skipping."
                              else
                                echo "Package ${PKG_NAME} not found. Uploading ${pkg}..."
                                twine upload --repository-url ${LOCAL_PYPI_SERVER}/ ${pkg}
                              fi
                            done
                        '''
                    }
                }
            }
        }

        // --- The following stages are now restored ---
        stage('Build Local Ansible Agent') {
            steps {
                script {
                    docker.build(AGENT_IMAGE_NAME, "./ci")
                }
            }
        }

        stage('Lint Ansible Code') {
            agent { docker { image AGENT_IMAGE_NAME } }
            steps {
                checkout scm
                sh "ansible-lint ."
            }
        }

        stage('Check Ansible Playbooks') {
            agent { docker { image AGENT_IMAGE_NAME } }
            steps {
                checkout scm
                sh "ansible-playbook -i inventory/staging.ini playbooks/main.yml --check"
            }
        }
    }
    
    // --- The post block is now restored ---
    post {
        always {
            sh "docker rmi ${AGENT_IMAGE_NAME} || true"
        }
    }
}