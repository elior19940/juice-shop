pipeline {
    agent any

    environment {
        // Define the image tag for our local build
        LOCAL_IMAGE = "my-juice-shop:latest"
    }

    stages {
        stage('1. Checkout') {
            steps {
                // Pulling the latest source code from your GitHub fork
                checkout scm
            }
        }

        stage('2. Secrets Scanning (Gitleaks)') {
            steps {
                // Running Gitleaks as root to ensure it can access all files in the mounted volume
                // We use || true because Juice Shop contains intentional secrets for training
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
                // Using auto-config to detect the best rules for a Node.js project
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
                // Software Composition Analysis: scanning package.json for known vulnerable libraries
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
                // Building the actual Docker image from the provided Dockerfile in the repository
                // This step represents the "Package" phase of the DevSecOps lifecycle
                sh "docker build -t ${LOCAL_IMAGE} ."
            }
        }

        stage('6. Container Image Scan (Trivy Image)') {
            steps {
                // Scanning the newly built OS image for vulnerabilities before deployment
                // This checks for CVEs in the base image (Alpine/Debian) and installed OS packages
                sh "docker run --rm -u 0:0 aquasec/trivy:latest image --severity HIGH,CRITICAL ${LOCAL_IMAGE}"
            }
        }

        stage('7. DAST (OWASP ZAP)') {
            steps {
                script {
                    // Start the vulnerable application container in the background
                    sh "docker run -d --name juice-shop-dast -p 3000:3000 ${LOCAL_IMAGE}"
                    
                    // Wait for the server to initialize properly (30 seconds)
                    sh "sleep 30"
                    
                    // Run a dynamic scan (active attack) against the running instance
                    // We map the workspace to save the HTML report
                    sh "docker run --rm --network host -v ${WORKSPACE}:/zap/wrk/:rw owasp/zap2docker-stable zap-baseline.py -t http://localhost:3000 -r zap_report.html || true"
                    
                    // Cleanup: Stop and remove the temporary test container
                    sh "docker stop juice-shop-dast && docker rm juice-shop-dast"
                }
            }
        }
    }

    post {
        always {
            // Archive the ZAP report as a build artifact to view it in Jenkins UI
            archiveArtifacts artifacts: 'zap_report.html', fingerprint: true
            // Clean workspace to free up disk space on the WSL host
            cleanWs()
        }
    }
}