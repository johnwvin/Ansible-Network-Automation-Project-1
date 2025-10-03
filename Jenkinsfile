pipeline {
    // Start on any available agent that has Docker installed.
    agent any

    stages {
        // --- STAGE 1: Build the Environment ---
        stage('Build Agent Image') {
            steps {
                script {
                    // This step reads your ./ci/Dockerfile and builds an image.
                    // We will name it 'ansible-lint-agent-local' just for this pipeline run.
                    echo "Building the agent image from the Dockerfile..."
                    def customImage = docker.build("ansible-lint-agent-local", "./ci")
                }
            }
        }

        // --- STAGE 2: Use the Environment We Just Built ---
        stage('Run Linting in Custom Agent') {
            // This is the key: this stage runs on a different, dynamic agent.
            // It uses the image we just built in the previous stage.
            agent {
                docker { image "ansible-lint-agent-local" }
            }
            steps {
                // We must check out the code again inside this new container
                // so the linter has something to scan.
                checkout scm
                
                echo "Running ansible-lint..."
                sh 'ansible-lint .'
            }
        }
    }
    
    // This optional block will clean up the temporary Docker image
    // after the pipeline is finished, keeping your agent machine clean.
    post {
        always {
            script {
                echo "Cleaning up temporary Docker image..."
                sh "docker rmi ansible-lint-agent-local || true"
            }
        }
    }
}