pipeline {
    agent any
    environment {
        LOCAL_IMAGE = "my-juice-shop:latest"
    }
    stages {
        stage('Security Analysis (SAST & Secrets)') {
            parallel {
                stage('Gitleaks') {
                    agent { docker { image 'zricethezav/gitleaks:latest' } }
                    steps { sh 'gitleaks detect --source . --verbose' }
                }
                stage('Semgrep') {
                    agent { docker { image 'semgrep/semgrep:latest' } }
                    steps { sh 'semgrep scan --config auto --error' }
                }
            }
        }
        stage('SCA - Dependency Scan (Trivy)') {
            agent { docker { image 'aquasec/trivy:latest' } }
            steps { sh 'trivy fs --severity HIGH,CRITICAL .' }
        }
        stage('Build Image') {
            steps { sh "docker build -t ${LOCAL_IMAGE} ." }
        }
        stage('Container Scan (Trivy)') {
            agent { docker { image 'aquasec/trivy:latest' } }
            steps { sh "trivy image --severity HIGH,CRITICAL ${LOCAL_IMAGE}" }
        }
        stage('DAST (OWASP ZAP)') {
            steps {
                script {
                    sh "docker run -d --name juice-shop-dast -p 3000:3000 ${LOCAL_IMAGE}"
                    sh "sleep 30"
                    sh "docker run --rm --network host -v \$(pwd):/zap/wrk/:rw owasp/zap2docker-stable zap-baseline.py -t http://localhost:3000 -r zap_report.html || true"
                    sh "docker stop juice-shop-dast && docker rm juice-shop-dast"
                }
            }
        }
    }
    post {
        always {
            archiveArtifacts artifacts: 'zap_report.html', fingerprint: true
            cleanWs()
        }
    }
}
