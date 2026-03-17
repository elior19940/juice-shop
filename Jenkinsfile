pipeline {
    agent any

    environment {
        // The unique tag for our locally built image
        LOCAL_IMAGE = "my-juice-shop:latest"
    }

    stages {
        stage('1. Checkout') {
            steps {
                // Fetching the source code from your GitHub fork
                checkout scm
            }
        }

        stage('2. Secrets Scanning (Gitleaks)') {
            steps {
                // Scanning the repository for leaked API keys or passwords
                // Using root user (-u 0:0) to ensure filesystem access
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
                // Static Application Security Testing to find code-level vulnerabilities
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
                // Scanning the project's filesystem for vulnerable libraries
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
                // Packaging the application into a Docker image
                sh "docker build -t ${LOCAL_IMAGE} ."
            }
        }

        stage('6. Container Image Scan (Trivy Image)') {
            steps {
                // IMPORTANT: We must mount the docker.sock so Trivy can see local images
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
                    // Running a temporary instance of the app for dynamic testing
                    sh "docker run -d --name juice-shop-dast -p 3000:3000 ${LOCAL_IMAGE}"
                    
                    // Allow the application 30 seconds to fully start up
                    sh "sleep 30"
                    
                    // Executing active attack against the running instance
                    sh "docker run --rm --network host -v ${WORKSPACE}:/zap/wrk/:rw owasp/zap2docker-stable zap-baseline.py -t http://localhost:3000 -r zap_report.html || true"
                    
                    // Cleanup test resources
                    sh "docker stop juice-shop-dast && docker rm juice-shop-dast"
                }
            }
        }
    }

    post {
        always {
            // Archiving the security report to be viewed in the Jenkins UI
            archiveArtifacts artifacts: 'zap_report.html', fingerprint: true
            // Cleaning workspace to prevent disk space issues
            cleanWs()
        }
    }
}