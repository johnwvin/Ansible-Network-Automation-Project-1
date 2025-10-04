pipeline {
    // The main pipeline starts on any agent with Docker installed
    agent any

    environment {
        // Define addresses for the services running on the agent
        LOCAL_DOCKER_REGISTRY = "localhost:5000"
        LOCAL_PYPI_SERVER = "http://localhost:8080"
        
        // Define a unique name for the agent image we build in this pipeline
        AGENT_IMAGE_NAME = "local-ansible-project-agent:${env.BUILD_ID}"
    }

    stages {
        // --- STAGE 1: Verify and Populate Local Caches ---
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
                    // Use a temporary container to download/upload Python packages
                    docker.image('python:3.13-slim').inside {
                        sh "pip install -r ci/python_packages.txt"
                        sh "pip download -r ci/python_packages.txt -d ./packages"
                        // --skip-existing handles the "if not found" logic automatically
                        sh "twine upload --repository-url ${LOCAL_PYPI_SERVER}/ --skip-existing ./packages/*"
                    }
                }
            }
        }

        // --- STAGE 2: Build the Final Agent Image Using Local Caches ---
        stage('Build Local Ansible Agent') {
            steps {
                echo "Building the final agent image using local caches..."
                // This builds the Dockerfile you provided
                docker.build(AGENT_IMAGE_NAME, "./ci")
            }
        }

        // --- STAGE 3: Run Static Analysis (Lint) ---
        stage('Lint Ansible Code') {
            agent {
                // Use the image we just built in the previous stage
                docker { image AGENT_IMAGE_NAME }
            }
            steps {
                checkout scm
                sh "ansible-lint ."
            }
        }

        // --- STAGE 4: Run Ansible Configuration Check ---
        stage('Check Ansible Playbooks') {
            agent {
                // Use the same fresh image
                docker { image AGENT_IMAGE_NAME }
            }
            steps {
                checkout scm
                // Run ansible-playbook in check mode to verify syntax and logic
                // Update the inventory and playbook paths as needed for your project
                sh "ansible-playbook -i inventory/staging.ini playbooks/main.yml --check"
            }
        }
    }
    
    post {
        // This block runs at the end to clean up the temporary image
        always {
            echo "Cleaning up local Docker image: ${AGENT_IMAGE_NAME}"
            sh "docker rmi ${AGENT_IMAGE_NAME} || true"
        }
    }
}