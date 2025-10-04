pipeline {
    agent any

    environment {
        CACHE_DIR = "/home/jenkins-agent/agent/caches"
        PIPELINE_NETWORK = "ansible-project-net-${env.BUILD_ID}"
        AGENT_IMAGE_NAME = "local-ansible-project-agent:${env.BUILD_ID}"
        // We can now reuse this variable for both the agent and the containers
        LOCAL_DOCKER_REGISTRY = "localhost:5000"
    }

    stages {
        stage('Setup Services and Network') {
            steps {
                script {
                    sh "mkdir -p ${CACHE_DIR}/docker-registry ${CACHE_DIR}/pypi-packages"
                    sh "docker network create ${PIPELINE_NETWORK}"
                    
                    // *** THE FIRST FIX IS HERE ***
                    // Add -p 5000:5000 to publish the port to the agent host
                    sh """
                        docker run -d --name registry-server --network ${PIPELINE_NETWORK} \
                        -p 5000:5000 \
                        -v ${CACHE_DIR}/docker-registry:/var/lib/registry \
                        registry:2
                    """
                    sh """
                        docker run -d --name pypi-server --network ${PIPELINE_NETWORK} \
                        -v ${CACHE_DIR}/pypi-packages:/data/packages \
                        pypiserver/pypiserver:latest
                    """
                    echo "Waiting for services to start..."
                    sleep 5
                }
            }
        }

        stage('Sync Dependencies') {
            steps {
                script {
                    echo "--- Syncing Docker Images ---"
                    def dockerImages = readFile('ci/docker_images.txt').trim().split('\n')
                    dockerImages.each { imageName ->
                        sh "docker pull ${imageName}"
                        
                        // *** THE SECOND FIX IS HERE ***
                        // Tag and push to localhost, which is where the agent can find the published port.
                        sh "docker tag ${imageName} ${LOCAL_DOCKER_REGISTRY}/${imageName}"
                        sh "docker push ${LOCAL_DOCKER_REGISTRY}/${imageName}"
                    }

                    echo "\n--- Syncing Python Packages ---"
                    // This part remains the same as it runs inside a container on the private network
                    docker.image('python:3.13-slim').inside("--network ${PIPELINE_NETWORK} -u root") {
                        sh "pip install -r ci/python_packages.txt"
                        sh "pip download -r ci/python_packages.txt -d ./packages"
                        sh "twine upload --repository-url http://pypi-server:8080/ --skip-existing ./packages/*"
                    }
                }
            }
        }

        // ... The rest of the pipeline remains the same
        stage('Build Ansible Agent') { /* ... */ }
        stage('Lint and Check Playbooks') { /* ... */ }
    }
    post { /* ... */ }
}