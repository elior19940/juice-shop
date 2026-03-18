pipeline {
    agent any

    environment {
        // Defining the local image name that will be built and scanned
        LOCAL_IMAGE = "my-juice-shop:latest"
    }

    stages {
        stage('1. Checkout') {
            steps {
                // Pulling the latest source code from your repository
                checkout scm
            }
        }

        stage('2. Secrets Scanning (Gitleaks)') {
            steps {
                // Scanning for hardcoded secrets (API keys, passwords, etc.)
                // || true ensures the pipeline continues even if findings are found
                sh '''
                docker run --rm \
                    -v ${WORKSPACE}:/src \
                    -u 0:0 \
                    zricethezav/gitleaks:latest \
                    detect --source /src --verbose || true
                '''
            }
        }

        stage('3. SAST (Semgrep)') {
            steps {
                // Performing Static Application Security Testing on the source code
                sh '''
                docker run --rm \
                    -v ${WORKSPACE}:/src \
                    -u 0:0 \
                    semgrep/semgrep:latest \
                    semgrep scan --config auto --error /src || true
                '''
            }
        }

        stage('4. SCA - Dependency Scan (Trivy FS)') {
            steps {
                // Scanning the filesystem (package.json) for vulnerable dependencies
                sh '''
                docker run --rm \
                    -v ${WORKSPACE}:/src \
                    -u 0:0 \
                    aquasec/trivy:latest \
                    fs --severity HIGH,CRITICAL /src
                '''
            }
        }

        stage('5. Build Image from Source') {
            steps {
                // Building the Docker image locally. 
                // Using the local daemon on your Windows/WSL machine.
                sh "docker build -t ${LOCAL_IMAGE} ."
            }
        }

        stage('6. Container Image Scan (Trivy Image)') {
            steps {
                // CRITICAL FIX: Mounting the Docker socket to allow Trivy to scan local images
                // This connects the Trivy container to the host's Docker engine
                sh '''
                docker run --rm \
                    -v /var/run/docker.sock:/var/run/docker.sock \
                    -u 0:0 \
                    aquasec/trivy:latest \
                    image --severity HIGH,CRITICAL ${LOCAL_IMAGE}
                '''
            }
        }

        stage('7. DAST (OWASP ZAP)') {
            steps {
                script {
                    // Running the app container in the background for dynamic testing
                    sh "docker run -d --name juice-shop-dast -p 3000:3000 ${LOCAL_IMAGE}"
                    
                    // Giving the application 30 seconds to start its web server
                    sh "sleep 30"
                    
                    // Running ZAP baseline scan against the live application
                    // Using --network host so the ZAP container can reach localhost:3000
                    sh "docker run --rm --network host -v ${WORKSPACE}:/zap/wrk/:rw owasp/zap2docker-stable zap-baseline.py -t http://localhost:3000 -r zap_report.html || true"
                    
                    // Cleanup: Stopping and removing the test container
                    sh "docker stop juice-shop-dast && docker rm juice-shop-dast"
                }
            }
        }
    }

    post {
        always {
            // Archiving the generated security report to be viewed in Jenkins UI
            archiveArtifacts artifacts: 'zap_report.html', fingerprint: true
            // Cleaning up the workspace to save disk space
            cleanWs()
        }
    }
}