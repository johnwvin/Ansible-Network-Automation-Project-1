pipeline {
    agent any

    environment {
        CACHE_DIR = "/home/jenkins-agent/agent/caches"
        PIPELINE_NETWORK = "ansible-project-net-${env.BUILD_ID}"
        AGENT_IMAGE_NAME = "local-ansible-project-agent:${env.BUILD_ID}"
    }

    stages {
        stage('Setup Services and Network') {
            steps {
                script {
                    sh "mkdir -p ${CACHE_DIR}/docker-registry ${CACHE_DIR}/pypi-packages"
                    sh "docker network create ${PIPELINE_NETWORK}"
                    sh """
                        docker run -d --name registry-server --network ${PIPELINE_NETWORK} \
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
                    // *** THE FIX IS HERE ***
                    // This section now runs directly on the agent, which has Docker installed.
                    echo "--- Syncing Docker Images ---"
                    def dockerImages = readFile('ci/docker_images.txt').trim().split('\n')
                    dockerImages.each { imageName ->
                        sh "docker pull ${imageName}"
                        sh "docker tag ${imageName} registry-server:5000/${imageName}"
                        sh "docker push registry-server:5000/${imageName}"
                    }

                    // This section still runs inside the Python container for pip and twine.
                    echo "\n--- Syncing Python Packages ---"
                    docker.image('python:3.13-slim').inside("--network ${PIPELINE_NETWORK} -u root") {
                        sh "pip install -r ci/python_packages.txt"
                        sh "pip download -r ci/python_packages.txt -d ./packages"
                        sh "twine upload --repository-url http://pypi-server:8080/ --skip-existing ./packages/*"
                    }
                }
            }
        }

        stage('Build Ansible Agent') {
            steps {
                script {
                    docker.build(AGENT_IMAGE_NAME, "--network ${PIPELINE_NETWORK} ./ci")
                }
            }
        }

        stage('Lint and Check Playbooks') {
            agent {
                docker { image AGENT_IMAGE_NAME }
            }
            steps {
                checkout scm
                sh "ansible-lint ."
                sh "ansible-playbook -i inventory/staging.ini playbooks/main.yml --check"
            }
        }
    }

    post {
        always {
            script {
                echo "--- Cleaning up pipeline infrastructure ---"
                sh "docker rm -f registry-server pypi-server || true"
                sh "docker rmi ${AGENT_IMAGE_NAME} || true"
                sh "docker network rm ${PIPELINE_NETWORK} || true"
            }
        }
    }
}