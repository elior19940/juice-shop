# DevSecOps Pipeline: Securing OWASP Juice Shop 🛡️

A comprehensive CI/CD pipeline built with **Jenkins**, designed to integrate security at every stage of the development lifecycle (Shift-Left Security). This project uses the **OWASP Juice Shop**—a deliberately insecure web application—as a target to demonstrate modern security automation.

## 🏗️ Architecture & Pipeline Flow
The pipeline is designed to be linear and robust, running on **WSL2** with a **Docker-out-of-Docker (DooD)** configuration. It ensures that no code is deployed without passing a rigorous battery of security tests.

1. **Static Analysis (SAST/Secrets):** Gitleaks & Semgrep.
2. **Dependency Scanning (SCA):** Trivy FS.
3. **Container Security:** Trivy Image.
4. **Dynamic Analysis (DAST):** OWASP ZAP (Active Baseline Scan).

## 🛠️ Security Stack & Findings
The pipeline successfully identified multiple critical vulnerabilities during its execution:

* **Secrets Scanning (Gitleaks):** Detected **11 hardcoded secrets**, including AWS tokens and private API keys.
* **SAST (Semgrep):** Identified **53 code-level security issues**, such as the use of dangerous JavaScript functions and insecure logging practices.
* **SCA (Trivy FS):** Flagged **Critical & High vulnerabilities** in third-party libraries like `body-parser` and `cookie-parser`.
* **DAST (OWASP ZAP):** Performed an active scan on the running container, generating a detailed report with **Medium-risk findings** like missing Anti-CSRF tokens.

## 🚀 Key Challenges Overcome
* **Virtualized Infrastructure:** Resolved complex path-mapping issues between Jenkins containers and the WSL2 host to ensure scanners could "see" the source code.
* **Docker Socket Integration:** Implemented secure mounting of `/var/run/docker.sock` to allow containerized scanners to inspect local Docker images.

## 📊 Results
The final pipeline provides a "Green" status only when all technical stages pass, but generates comprehensive artifacts (HTML reports) for security teams to review.

---
*This project was built as part of my journey to becoming a Security Operations Engineer.*
