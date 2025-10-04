pipeline {
    agent any

    environment {
        LOCAL_DOCKER_REGISTRY = "localhost:5000"
        LOCAL_PYPI_SERVER = "http://localhost:8080"
        AGENT_IMAGE_NAME = "local-ansible-project-agent:${env.BUILD_ID}"
    }

    stages {
        // --- STAGE 1: Create and Ensure Local Services are Running ---
        stage('Setup and Ensure Local Services') {
            steps {
                script {
                    echo "Checking for and creating local service configuration if needed..."
                    sh '''
                        #!/bin/bash
                        SERVICE_DIR="/opt/jenkins-services"
                        COMPOSE_FILE="${SERVICE_DIR}/docker-compose.yml"

                        # --- Step 1: Check if the main directory exists. If not, MAKE IT. ---
                        if [ ! -d "${SERVICE_DIR}" ]; then
                          echo "Service directory ${SERVICE_DIR} not found. Creating it now..."
                          sudo mkdir -p "${SERVICE_DIR}/docker-registry-data"
                          sudo mkdir "${SERVICE_DIR}/pypi-server-packages"
                        fi

                        # --- Step 2: Check if the docker-compose.yml file exists. If not, MAKE IT. ---
                        if [ ! -f "${COMPOSE_FILE}" ]; then
                          echo "Compose file ${COMPOSE_FILE} not found. Creating it now..."
                          # Use a 'here document' to write the entire file content
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
                        
                        # --- Step 3: Now that we guarantee the config exists, check if services are running. ---
                        cd ${SERVICE_DIR}
                        RUNNING_SERVICES=$(sudo docker-compose ps | grep "Up" | wc -l)
                        
                        if [ "$RUNNING_SERVICES" -lt 2 ]; then
                          echo "One or more local services are down. Starting them now..."
                          sudo docker-compose up -d
                          sleep 5
                        else
                          echo "All local services are already running."
                        fi
                    '''
                }
            }
        }

        // --- The rest of the pipeline remains the same ---
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
    
    post {
        always {
            sh "docker rmi ${AGENT_IMAGE_NAME} || true"
        }
    }
}