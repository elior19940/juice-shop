pipeline {
    agent any

    environment {
        LOCAL_IMAGE = "my-juice-shop:latest"
        // Using the GitHub Container Registry for ZAP as it is more reliable than Docker Hub
        ZAP_IMAGE = "ghcr.io/zaproxy/zaproxy:stable"
    }

    stages {
        stage('1. Prepare Environment') {
            steps {
                script {
                    // PRO TIP: Discovering the actual host path of the workspace 
                    // This fixes the "empty scan" issue in Docker-out-of-Docker setups
                    try {
                        env.HOST_WORKSPACE = sh(script: "docker inspect \$(hostname) -f '{{range .Mounts}}{{if eq .Destination \"/var/jenkins_home\"}}{{.Source}}{{end}}{{end}}'", returnStdout: true).trim() + "/workspace/${JOB_NAME}"
                        echo "Discovered Host Workspace: ${env.HOST_WORKSPACE}"
                    } catch (Exception e) {
                        echo "Failed to discover host path, falling back to internal path"
                        env.HOST_WORKSPACE = "${WORKSPACE}"
                    }
                }
            }
        }

        stage('2. Checkout') {
            steps {
                checkout scm
            }
        }

        stage('3. Secrets Scanning (Gitleaks)') {
            steps {
                // Using --no-git to ensure it scans files even if the .git folder mount is tricky
                sh """
                docker run --rm \
                    -v ${env.HOST_WORKSPACE}:/src \
                    -u 0:0 \
                    zricethezav/gitleaks:latest \
                    detect --source /src --verbose --no-git || true
                """
            }
        }

        stage('4. SAST (Semgrep)') {
            steps {
                sh """
                docker run --rm \
                    -v ${env.HOST_WORKSPACE}:/src \
                    -u 0:0 \
                    semgrep/semgrep:latest \
                    semgrep scan --config auto --error /src || true
                """
            }
        }

        stage('5. SCA - Dependency Scan (Trivy FS)') {
            steps {
                // Now Trivy will find the language-specific files (package.json)
                sh """
                docker run --rm \
                    -v ${env.HOST_WORKSPACE}:/src \
                    -u 0:0 \
                    aquasec/trivy:latest \
                    fs --severity HIGH,CRITICAL /src
                """
            }
        }

        stage('6. Build Image from Source') {
            steps {
                sh "docker build -t ${LOCAL_IMAGE} ."
            }
        }

        stage('7. Container Image Scan (Trivy Image)') {
            steps {
                sh """
                docker run --rm \
                    -v /var/run/docker.sock:/var/run/docker.sock \
                    -u 0:0 \
                    aquasec/trivy:latest \
                    image --severity HIGH,CRITICAL ${LOCAL_IMAGE}
                """
            }
        }

        stage('8. DAST (OWASP ZAP)') {
            steps {
                script {
                    sh "docker run -d --name juice-shop-dast -p 3000:3000 ${LOCAL_IMAGE}"
                    sh "sleep 30"
                    
                    // Fixed the pull issue by using the full GHCR path
                    sh """
                    docker run --rm --network host \
                        -v ${env.HOST_WORKSPACE}:/zap/wrk/:rw \
                        ${env.ZAP_IMAGE} zap-baseline.py \
                        -t http://localhost:3000 -r zap_report.html || true
                    """
                    
                    sh "docker stop juice-shop-dast && docker rm juice-shop-dast"
                }
            }
        }
    }

    post {
        always {
            // Archiving artifacts - this time they should exist!
            archiveArtifacts artifacts: 'zap_report.html', allowEmptyArchive: true, fingerprint: true
            cleanWs()
        }
    }
}