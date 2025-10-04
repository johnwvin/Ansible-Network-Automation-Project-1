pipeline {
    agent any

    environment {
        LOCAL_DOCKER_REGISTRY = "localhost:5000"
        LOCAL_PYPI_SERVER = "http://localhost:8080"
        AGENT_IMAGE_NAME = "local-ansible-project-agent:${env.BUILD_ID}"
    }

    stages {
        stage('Setup and Ensure Local Services') {
            // ... (This stage remains the same)
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
                        RUNNING_SERVICES=$(sudo docker compose ps | grep "Up" | wc -l)
                        if [ "$RUNNING_SERVICES" -lt 2 ]; then
                          sudo docker compose up -d
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
                    // *** THE FIX IS HERE ***
                    // This script now manually checks if packages exist before uploading.
                    docker.image('python:3.13-slim').inside("-u root") {
                        // First, download all required packages into a directory
                        sh "pip install -r ci/python_packages.txt"
                        sh "pip download -r ci/python_packages.txt -d ./packages"
                        
                        // Now, intelligently upload them
                        sh '''
                            #!/bin/bash
                            # Loop through every downloaded package file (.whl)
                            for pkg in ./packages/*.whl; do
                              # Extract just the package name (e.g., ansible-lint)
                              PKG_NAME=$(basename ${pkg} | cut -d- -f1)

                              echo "Checking for package: ${PKG_NAME} on local server..."
                              # Search the local server for the package name. Grep for the name case-insensitively.
                              # If grep finds it, the exit code is 0 (success).
                              if pip search --index ${LOCAL_PYPI_SERVER}/simple ${PKG_NAME} | grep -i ${PKG_NAME}; then
                                echo "Package ${PKG_NAME} already exists on the local server. Skipping."
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

        // --- The rest of the pipeline remains the same ---
        stage('Build Local Ansible Agent') { /* ... */ }
        stage('Lint Ansible Code') { /* ... */ }
        stage('Check Ansible Playbooks') { /* ... */ }
    }
    
    post { /* ... */ }
}