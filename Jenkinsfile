pipeline {
    agent any

    environment {
        LOCAL_IMAGE = "my-juice-shop:latest"
        // Using the updated official image from GHCR for ZAP
        ZAP_IMAGE = "ghcr.io/zaproxy/zaproxy:stable"
    }

    stages {
        stage('1. Checkout') {
            steps {
                checkout scm
            }
        }

        stage('2. Secrets Scanning (Gitleaks)') {
            steps {
                // Mapping the current directory to /src and running detect
                // Using --no-git if the .git folder mapping is problematic in Docker-out-of-Docker
                sh '''
                docker run --rm \
                    -v ${WORKSPACE}:/src \
                    -u 0:0 \
                    zricethezav/gitleaks:latest \
                    detect --source /src --verbose --no-git || true
                '''
            }
        }

        stage('3. SAST (Semgrep)') {
            steps {
                sh '''
                docker run --rm \
                    -v ${WORKSPACE}:/src \
                    -u 0:0 \
                    semgrep/semgrep:latest \
                    semgrep scan --config auto --error /src || true
                '''
            }
        }

        stage('4. SCA (Trivy FS)') {
            steps {
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
                sh "docker build -t ${LOCAL_IMAGE} ."
            }
        }

        stage('6. Container Image Scan (Trivy Image)') {
            steps {
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
                    sh "docker run -d --name juice-shop-dast -p 3000:3000 ${LOCAL_IMAGE}"
                    sh "sleep 30"
                    
                    // Fixed: Using the updated ZAP_IMAGE and proper mapping
                    sh """
                    docker run --rm --network host \
                        -v ${WORKSPACE}:/zap/wrk/:rw \
                        ${ZAP_IMAGE} zap-baseline.py \
                        -t http://localhost:3000 -r zap_report.html || true
                    """
                    
                    sh "docker stop juice-shop-dast && docker rm juice-shop-dast"
                }
            }
        }
    }

    post {
        always {
            // Archive artifacts only if they exist to avoid confusion
            archiveArtifacts artifacts: 'zap_report.html', allowEmptyArchive: true, fingerprint: true
            cleanWs()
        }
    }
}