pipeline {
    agent any

    environment {
        // Name of the image to be built from the Juice Shop source code
        LOCAL_IMAGE = "my-juice-shop:latest"
    }

    stages {
        stage('1. Checkout') {
            steps {
                // Pulling the code from your repository fork
                checkout scm
            }
        }

        stage('2. Secrets Scanning (Gitleaks)') {
            agent { 
                docker { 
                    image 'zricethezav/gitleaks:latest'
                    // Using Root user to prevent container crashes on WSL environments
                    args '-v ${WORKSPACE}:/src -u 0:0'
                } 
            }
            steps { 
                // Scanning for hardcoded secrets. '|| true' allows the pipeline to continue despite findings
                sh 'gitleaks detect --source /src --verbose || true' 
            }
        }

        stage('3. SAST (Semgrep)') {
            agent { 
                docker { 
                    image 'semgrep/semgrep:latest'
                    args '-v ${WORKSPACE}:/src -u 0:0'
                } 
            }
            steps { 
                // Static Analysis Security Testing (SAST). Points the scanner to the mapped /src directory
                sh 'semgrep scan --config auto --error /src || true' 
            }
        }

        stage('4. SCA - Dependency Scan (Trivy FS)') {
            agent { 
                docker { 
                    image 'aquasec/trivy:latest'
                    args '-v ${WORKSPACE}:/src -u 0:0'
                } 
            }
            steps { 
                // Software Composition Analysis (SCA) - Scanning 3rd party libraries (e.g., package.json) for known vulnerabilities
                sh 'trivy fs --severity HIGH,CRITICAL /src' 
            }
        }

        stage('5. Build Image from Source') {
            steps { 
                // Building the application Docker image using the Dockerfile in the repository
                sh "docker build -t ${LOCAL_IMAGE} ." 
            }
        }

        stage('6. Container Image Scan (Trivy Image)') {
            agent { 
                docker { 
                    image 'aquasec/trivy:latest'
                } 
            }
            steps { 
                // Scanning the final image (OS layers and binaries) for vulnerabilities before execution
                sh "trivy image --severity HIGH,CRITICAL ${LOCAL_IMAGE}" 
            }
        }

        stage('7. DAST (OWASP ZAP)') {
            steps {
                script {
                    // Running the application in the background to perform active security testing (DAST)
                    sh "docker run -d --name juice-shop-dast -p 3000:3000 ${LOCAL_IMAGE}"
                    
                    // Brief pause to ensure the server is fully up and running
                    sh "sleep 30"
                    
                    // Executing Dynamic Analysis (active attack) against the running website
                    sh "docker run --rm --network host -v \$(pwd):/zap/wrk/:rw owasp/zap2docker-stable zap-baseline.py -t http://localhost:3000 -r zap_report.html || true"
                    
                    // Cleaning up the temporary test container
                    sh "docker stop juice-shop-dast && docker rm juice-shop-dast"
                }
            }
        }
    }

    post {
        always {
            // Archiving the DAST report as a Jenkins Artifact for review
            archiveArtifacts artifacts: 'zap_report.html', fingerprint: true
            // Cleaning the workspace to prevent storage buildup (crucial for WSL/local runners)
            cleanWs()
        }
    }
}