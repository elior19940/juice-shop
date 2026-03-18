pipeline {
    agent any

    environment {
        // Tag for the locally built image
        LOCAL_IMAGE = "my-juice-shop:latest"
    }

    stages {
        stage('1. Checkout') {
            steps {
                // Pull source code from GitHub
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
                // Static Analysis to find code vulnerabilities
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
                // Scanning filesystem (package.json) for vulnerable dependencies
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
                // Building the Docker image locally. This takes time in WSL.
                sh "docker build -t ${LOCAL_IMAGE} ."
            }
        }

        stage('6. Container Image Scan (Trivy Image)') {
            steps {
                // Mounting docker.sock allows Trivy to talk to the host's Docker engine
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
                    // Start the application in the background
                    sh "docker run -d --name juice-shop-dast -p 3000:3000 ${LOCAL_IMAGE}"
                    
                    // Wait 30 seconds for the web server to initialize
                    sh "sleep 30"
                    
                    // Attack the running application to find runtime flaws
                    // UPDATED: Using the new official image 'zaproxy/zaproxy:stable'
                    sh "docker run --rm --network host -v ${WORKSPACE}:/zap/wrk/:rw zaproxy/zaproxy:stable zap-baseline.py -t http://localhost:3000 -r zap_report.html || true"
                    
                    // Cleanup testing container
                    sh "docker stop juice-shop-dast && docker rm juice-shop-dast"
                }
            }
        }
    }

    post {
        always {
            // Save the ZAP security report as a build artifact
            // It will be available only if the ZAP scan generates the file
            archiveArtifacts artifacts: 'zap_report.html', allowEmptyArchive: true, fingerprint: true
            // Clean workspace to free up disk space in WSL
            cleanWs()
        }
    }
}