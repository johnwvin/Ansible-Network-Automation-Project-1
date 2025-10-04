pipeline {
    // This pipeline must run on an agent where the jenkins user can control Docker
    agent any

    environment {
        // --- Configuration ---
        // Directory on the agent's filesystem for persistent caches
        CACHE_DIR = "/home/jenkins-agent/agent/caches"
        // Name for the private network this pipeline will create and use
        PIPELINE_NETWORK = "ansible-project-net-${env.BUILD_ID}"
        // Name for the final image we build to run our tests
        AGENT_IMAGE_NAME = "local-ansible-project-agent:${env.BUILD_ID}"
    }

    stages {
        // --- STAGE 1: Setup Temporary Infrastructure ---
        stage('Setup Services and Network') {
            steps {
                script {
                    // Create cache directories on the host if they don't exist
                    sh "mkdir -p ${CACHE_DIR}/docker-registry ${CACHE_DIR}/pypi-packages"
                    
                    // Create a private network for our services to communicate on
                    sh "docker network create ${PIPELINE_NETWORK}"

                    // Start the Docker Registry container on our private network
                    sh """
                        docker run -d --name registry-server --network ${PIPELINE_NETWORK} \
                        -v ${CACHE_DIR}/docker-registry:/var/lib/registry \
                        registry:2
                    """

                    // Start the PyPI Server container on our private network
                    sh """
                        docker run -d --name pypi-server --network ${PIPELINE_NETWORK} \
                        -v ${CACHE_DIR}/pypi-packages:/data/packages \
                        pypiserver/pypiserver:latest
                    """
                    
                    // Give the services a moment to initialize
                    echo "Waiting for services to start..."
                    sleep 5
                }
            }
        }

        // --- STAGE 2: Sync Dependencies to Caches ---
        stage('Sync Dependencies') {
            steps {
                script {
                    // This temporary container also needs to be on our private network to see the services
                    docker.image('python:3.13-slim').inside("--network ${PIPELINE_NETWORK} -u root") {
                        echo "--- Syncing Docker Images ---"
                        def dockerImages = readFile('ci/docker_images.txt').trim().split('\n')
                        dockerImages.each { imageName ->
                            sh "docker pull ${imageName}"
                            sh "docker tag ${imageName} registry-server:5000/${imageName}"
                            sh "docker push registry-server:5000/${imageName}"
                        }

                        echo "\n--- Syncing Python Packages ---"
                        sh "pip install -r ci/python_packages.txt"
                        sh "pip download -r ci/python_packages.txt -d ./packages"
                        sh "twine upload --repository-url http://pypi-server:8080/ --skip-existing ./packages/*"
                    }
                }
            }
        }

        // --- STAGE 3: Build Final Agent Image ---
        stage('Build Ansible Agent') {
            steps {
                script {
                    // Build our final agent using the caches populated in the previous stage
                    docker.build(AGENT_IMAGE_NAME, "--network ${PIPELINE_NETWORK} ./ci")
                }
            }
        }

        // --- STAGE 4 & 5: Run Checks ---
        stage('Lint and Check Playbooks') {
            agent {
                // Use the image we just built to run our tests
                docker { image AGENT_IMAGE_NAME }
            }
            steps {
                checkout scm
                sh "ansible-lint ."
                sh "ansible-playbook -i inventory/staging.ini playbooks/main.yml --check"
            }
        }
    }

    // --- FINAL STEP: Automatic Cleanup ---
    post {
        always {
            script {
                echo "--- Cleaning up pipeline infrastructure ---"
                // Stop and remove the service containers, ignoring errors if they're already gone
                sh "docker rm -f registry-server pypi-server || true"
                
                // Remove the locally built agent image
                sh "docker rmi ${AGENT_IMAGE_NAME} || true"
                
                // Remove the private network
                sh "docker network rm ${PIPELINE_NETWORK} || true"
            }
        }
    }
}