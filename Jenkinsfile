pipeline {
    agent any

    environment {
        // The name of the image we will build from the source code
        LOCAL_IMAGE = "my-juice-shop:latest"
    }

    stages {
        stage('Checkout') {
            steps {
                // Pulling the forked Juice Shop source code
                checkout scm
            }
        }

        stage('Security Analysis (Static)') {
            parallel {
                stage('Gitleaks (Secrets)') {
                    agent { 
                        docker { 
                            image 'zricethezav/gitleaks:latest'
                            // Fix: Mapping the workspace to /src inside the container
                            args '-v ${WORKSPACE}:/src'
                        } 
                    }
                    steps { 
                        // Scanning for leaked credentials in the source code
                        sh 'gitleaks detect --source /src --verbose' 
                    }
                }
                stage('Semgrep (SAST)') {
                    agent { 
                        docker { 
                            image 'semgrep/semgrep:latest'
                            // Fix: Required by Semgrep to see the code files
                            args '-v ${WORKSPACE}:/src'
                        } 
                    }
                    steps { 
                        // Static Analysis for security flaws in JavaScript
                        sh 'semgrep scan --config auto --error /src' 
                    }
                }
            }
        }

        stage('SCA - Dependency Scan (Trivy FS)') {
            agent { 
                docker { 
                    image 'aquasec/trivy:latest'
                    // Mapping the workspace to scan the package.json
                    args '-v ${WORKSPACE}:/src'
                } 
            }
            steps { 
                // Scanning for vulnerable third-party libraries
                sh 'trivy fs --severity HIGH,CRITICAL /src' 
            }
        }

        stage('Build Image from Source') {
            steps { 
                // Building the actual Juice Shop image using the local Dockerfile
                sh "docker build -t ${LOCAL_IMAGE} ." 
            }
        }

        stage('Container Image Scan (Trivy Image)') {
            agent { 
                docker { 
                    image 'aquasec/trivy:latest'
                    // No volume needed here as we scan the built image directly
                } 
            }
            steps { 
                // Scanning the newly built OS image for vulnerabilities
                sh "trivy image --severity HIGH,CRITICAL ${LOCAL_IMAGE}" 
            }
        }

        stage('DAST (OWASP ZAP)') {
            steps {
                script {
                    // 1. Run the application container in the background
                    sh "docker run -d --name juice-shop-dast -p 3000:3000 ${LOCAL_IMAGE}"
                    
                    // 2. Wait for the app to initialize
                    sh "sleep 30"
                    
                    // 3. Perform dynamic scan against the running instance
                    // We use --network host to allow ZAP to reach localhost:3000
                    sh "docker run --rm --network host -v \$(pwd):/zap/wrk/:rw owasp/zap2docker-stable zap-baseline.py -t http://localhost:3000 -r zap_report.html || true"
                    
                    // 4. Cleanup the test container
                    sh "docker stop juice-shop-dast && docker rm juice-shop-dast"
                }
            }
        }
    }

    post {
        always {
            // Archive the ZAP report so it can be viewed in the Jenkins UI
            archiveArtifacts artifacts: 'zap_report.html', fingerprint: true
            // Clean the workspace to save space on the WSL host
            cleanWs()
        }
    }
}