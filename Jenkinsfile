pipeline {
    agent any

    environment {
        CACHE_DIR = "/home/jenkins-agent/agent/caches"
        PIPELINE_NETWORK = "ansible-project-net-${env.BUILD_ID}"
        AGENT_IMAGE_NAME = "local-ansible-project-agent:${env.BUILD_ID}"
        LOCAL_DOCKER_REGISTRY = "localhost:5000"
        LOCAL_PYPI_SERVER = "http://localhost:8080"
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
                          echo "Service directory ${SERVICE_DIR} not found. Creating it now..."
                          sudo mkdir -p "${SERVICE_DIR}/docker-registry-data"
                          sudo mkdir "${SERVICE_DIR}/pypi-server-packages"
                        fi

                        if [ ! -f "${COMPOSE_FILE}" ]; then
                          echo "Compose file ${COMPOSE_FILE} not found. Creating it now..."
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
                        # *** THE FIX IS HERE ***
                        # Changed 'docker-compose' to 'docker compose'
                        RUNNING_SERVICES=$(sudo docker compose ps | grep "Up" | wc -l)
                        
                        if [ "$RUNNING_SERVICES" -lt 2 ]; then
                          echo "One or more local services are down. Starting them now..."
                          # And also here
                          sudo docker compose up -d
                          sleep 5
                        else
                          echo "All local services are already running."
                        fi
                    '''
                }
            }
        }

        // --- The rest of the pipeline remains the same ---
        stage('Sync Dependencies') { /* ... */ }
        stage('Build Local Ansible Agent') { /* ... */ }
        stage('Lint and Check Playbooks') { /* ... */ }
    }
    
    post { /* ... */ }
}