# DoD-Compliant CI/CD Pipeline: Implementation Example

[← Back to Main Document](README.md)

This document provides a complete implementation example of a DoD-compliant CI/CD pipeline using Jenkins and various security tools.

## Table of Contents

1. [Pipeline Overview](#1-pipeline-overview)
2. [Implementation Prerequisites](#2-implementation-prerequisites)
3. [Complete Pipeline Code](#3-complete-pipeline-code)
4. [Stage Implementations](#4-stage-implementations)
5. [Tool Configurations](#5-tool-configurations)
6. [Deployment Considerations](#6-deployment-considerations)

## 1. Pipeline Overview

This implementation example demonstrates a complete CI/CD pipeline that incorporates all the security controls required by the DoD Cloud Security Playbook, including:

- Multi-tier code review process
- Comprehensive security testing
- Infrastructure validation
- Container security
- Compliance verification
- Secure deployment practices
- Runtime protection

## 2. Implementation Prerequisites

### 2.1 Required Tools

| Category | Tools |
|----------|-------|
| **CI/CD Platform** | Jenkins, Jenkins Pipelines |
| **Source Control** | Git with signed commits |
| **Code Review** | Custom review verification tool |
| **SAST** | SonarQube, Fortify SCA |
| **DAST** | OWASP ZAP, Burp Suite Enterprise |
| **SCA** | OWASP Dependency Check |
| **Container Security** | Trivy, Clair |
| **IaC Security** | Checkov, CloudSploit |
| **Compliance** | Custom compliance validators |
| **Deployment** | Blue-green deployment tools |
| **Monitoring** | Security monitoring solutions |

### 2.2 Environment Configuration

1. **Jenkins Setup**
   - Hardened Jenkins instance
   - Pipeline shared libraries for security functions
   - Credentials management via Jenkins Credentials Plugin
   - Role-based access control

2. **Security Tool Integration**
   - Tool integrations configured in Jenkins
   - API keys and credentials stored securely
   - Scan result parsing plugins

3. **Pipeline Infrastructure**
   - Isolated build agents
   - Network segmentation for build environments
   - Separate environments for testing and staging

4. **Access Controls**
   - Pipeline role-based access
   - Approval gates configured
   - Authentication with DoD-approved methods

## 3. Complete Pipeline Code

Below is a complete example of a Jenkinsfile implementing a DoD-compliant CI/CD pipeline:

```groovy
pipeline {
    agent none

    options {
        timestamps()
        timeout(time: 2, unit: 'HOURS')
        disableConcurrentBuilds()
    }

    environment {
        // Use credential store for sensitive variables
        DOCKER_REGISTRY_CREDENTIALS = credentials('docker-registry-credentials')
        ARTIFACT_REPOSITORY_CREDENTIALS = credentials('artifact-repo-credentials')
        SCAN_CREDENTIALS = credentials('security-scanner-credentials')

        // Environment variables
        APPLICATION_NAME = 'dod-secure-application'
        VERSION = "${BUILD_NUMBER}"
        BUILD_TIMESTAMP = sh(script: 'date +%Y%m%d%H%M%S', returnStdout: true).trim()
        GIT_COMMIT_HASH = sh(script: 'git rev-parse HEAD', returnStdout: true).trim()

        // Security thresholds
        CRITICAL_VULN_THRESHOLD = 0
        HIGH_VULN_THRESHOLD = 0
        MEDIUM_VULN_THRESHOLD = 5
    }

    stages {
        stage('Initialize') {
            agent {
                label 'secure-agent'
            }
            steps {
                // Initialize build environment
                sh 'security-environment-validator'
                sh 'setup-secure-workspace'

                // Log build start with security context
                sh 'log-secure-build-start --application=${APPLICATION_NAME} --version=${VERSION}'
            }
        }

        stage('Code Checkout') {
            agent {
                label 'secure-agent'
            }
            steps {
                // Verify code signatures
                sh 'git verify-commit HEAD'

                // Scan for secrets
                sh 'detect-secrets scan'

                // Code checkout with commit signature validation
                checkout([$class: 'GitSCM',
                    branches: [[name: '*/main']],
                    extensions: [[$class: 'CloneOption',
                        noTags: false,
                        shallow: false,
                        depth: 0,
                        reference: '']],
                    userRemoteConfigs: [[
                        url: 'https://github.com/organization/repository.git',
                        refspec: '+refs/heads/main:refs/remotes/origin/main'
                    ]]
                ])

                // Generate code source attestation
                sh 'generate-source-attestation --commit=${GIT_COMMIT_HASH}'
            }
        }

        stage('Review Verification') {
            agent {
                label 'secure-agent'
            }
            steps {
                // Verify peer review
                script {
                    def peerReviewStatus = sh(script: 'code-review-status-checker --level=peer', returnStdout: true).trim()
                    if (peerReviewStatus != 'APPROVED') {
                        error "Peer review has not been completed or approved"
                    }

                    // Record peer review metadata
                    sh 'record-review-metadata --level=peer --pipeline-run=${BUILD_NUMBER}'
                }

                // Verify security gatekeeper review
                script {
                    def gatekeeperReviewStatus = sh(script: 'code-review-status-checker --level=security-gatekeeper', returnStdout: true).trim()
                    if (gatekeeperReviewStatus != 'APPROVED') {
                        error "Security Gatekeeper review has not been completed or approved"
                    }

                    // Record gatekeeper review metadata
                    sh 'record-review-metadata --level=security-gatekeeper --pipeline-run=${BUILD_NUMBER}'
                }

                // Verify Mission Owner approval
                script {
                    def moApprovalStatus = sh(script: 'code-review-status-checker --level=mission-owner', returnStdout: true).trim()
                    if (moApprovalStatus != 'APPROVED') {
                        error "Mission Owner approval has not been granted"
                    }

                    // Record MO approval metadata
                    sh 'record-review-metadata --level=mission-owner --pipeline-run=${BUILD_NUMBER}'

                    // Generate approval attestation
                    sh 'generate-approval-attestation --signers=${APPROVERS} --commit=${GIT_COMMIT}'
                }
            }
        }

        stage('Security Analysis') {
            parallel {
                stage('SAST') {
                    agent {
                        label 'secure-agent-sast'
                    }
                    steps {
                        // Static Application Security Testing
                        sh 'sonarqube-scanner'
                        sh 'fortify-sca'

                        // Software Composition Analysis
                        sh 'dependency-check'

                        // Evaluate results
                        script {
                            def sastResults = readFile('sast-results.json')
                            if (sastResults.contains('"severity": "CRITICAL"')) {
                                error "Critical security vulnerabilities found in SAST scan"
                            }
                            if (sastResults.contains('"severity": "HIGH"')) {
                                error "High security vulnerabilities found in SAST scan"
                            }

                            // Generate SAST attestation
                            sh 'generate-sast-attestation --scan-id=${SAST_SCAN_ID}'
                        }
                    }
                    post {
                        always {
                            // Archive SAST results
                            archiveArtifacts artifacts: 'sast-results/**/*'
                        }
                    }
                }

                stage('Secrets Detection') {
                    agent {
                        label 'secure-agent'
                    }
                    steps {
                        // Detect secrets in code
                        sh 'detect-secrets scan'
                        sh 'trufflehog .'

                        // Evaluate results
                        script {
                            def secretsResults = readFile('secrets-results.json')
                            if (secretsResults.contains('"severity": "HIGH"')) {
                                error "Potential secrets found in code"
                            }
                        }
                    }
                    post {
                        always {
                            // Archive secrets detection results
                            archiveArtifacts artifacts: 'secrets-results/**/*'
                        }
                    }
                }

                stage('License Compliance') {
                    agent {
                        label 'secure-agent'
                    }
                    steps {
                        // Check license compliance
                        sh 'license-checker --production'

                        // Evaluate results
                        script {
                            def licenseResults = readFile('license-results.json')
                            if (licenseResults.contains('"status": "VIOLATION"')) {
                                error "License compliance violations detected"
                            }
                        }
                    }
                    post {
                        always {
                            // Archive license compliance results
                            archiveArtifacts artifacts: 'license-results/**/*'
                        }
                    }
                }
            }
        }

        stage('Build') {
            agent {
                label 'secure-agent-build'
            }
            steps {
                // Verify build environment
                sh 'verify-build-environment'

                // Signed build process
                sh 'mvn clean package -Dgpg.sign=true'

                // Verify build integrity
                sh 'verify-build-signatures'

                // Build container image
                sh 'docker build -t ${APPLICATION_NAME}:${VERSION} .'

                // Sign container image
                sh 'cosign sign ${APPLICATION_NAME}:${VERSION}'

                // Generate build attestation
                sh 'generate-build-attestation --artifact-id=${APPLICATION_NAME}:${VERSION}'
            }
            post {
                success {
                    // Archive build artifacts
                    archiveArtifacts artifacts: 'target/*.jar,target/*.sig'
                }
            }
        }

        stage('Container Security') {
            agent {
                label 'secure-agent'
            }
            steps {
                // Container image scanning
                sh 'trivy image ${APPLICATION_NAME}:${VERSION}'
                sh 'clair-scanner ${APPLICATION_NAME}:${VERSION}'

                // Image hardening verification
                sh 'container-compliance-scanner ${APPLICATION_NAME}:${VERSION}'

                // Evaluate results
                script {
                    def containerScanResults = readFile('container-scan-results.json')
                    if (containerScanResults.contains('"severity": "CRITICAL"')) {
                        error "Critical vulnerabilities found in container"
                    }
                    if (containerScanResults.contains('"severity": "HIGH"')) {
                        error "High vulnerabilities found in container"
                    }

                    // Generate container security attestation
                    sh 'generate-container-attestation --image=${APPLICATION_NAME}:${VERSION}'
                }
            }
            post {
                always {
                    // Archive container scan results
                    archiveArtifacts artifacts: 'container-scan-results/**/*'
                }
            }
        }

        stage('Infrastructure Validation') {
            agent {
                label 'secure-agent'
            }
            steps {
                // Infrastructure as Code scanning
                sh 'terraform validate'
                sh 'checkov -d terraform/'
                sh 'cloudsploit scan'

                // Evaluate results
                script {
                    def iacScanResults = readFile('iac-scan-results.json')
                    if (iacScanResults.contains('"severity": "HIGH"')) {
                        error "Critical misconfigurations found in infrastructure code"
                    }

                    // Generate IaC security attestation
                    sh 'generate-iac-attestation --scan-id=${IAC_SCAN_ID}'
                }
            }
            post {
                always {
                    // Archive IaC scan results
                    archiveArtifacts artifacts: 'iac-scan-results/**/*'
                }
            }
        }

        stage('Deploy to Test') {
            agent {
                label 'secure-agent-deploy'
            }
            steps {
                // Deploy to isolated test environment
                sh 'terraform apply -auto-approve -var="version=${VERSION}"'

                // Verify deployment
                sh 'deployment-validator --environment=test --version=${VERSION}'

                // Apply runtime protection
                sh 'runtime-protection-enabler --environment=test'
            }
        }

        stage('DAST') {
            agent {
                label 'secure-agent-dast'
            }
            steps {
                // Dynamic Application Security Testing
                sh 'owasp-zap-scan https://test-${APPLICATION_NAME}.example.com'
                sh 'burp-suite-enterprise-scan https://test-${APPLICATION_NAME}.example.com'

                // Evaluate results
                script {
                    def dastResults = readFile('dast-results.json')
                    if (dastResults.contains('"severity": "HIGH"')) {
                        error "Critical security vulnerabilities found in DAST scan"
                    }

                    // Generate DAST attestation
                    sh 'generate-dast-attestation --scan-id=${DAST_SCAN_ID}'
                }
            }
            post {
                always {
                    // Archive DAST results
                    archiveArtifacts artifacts: 'dast-results/**/*'
                }
            }
        }

        stage('Security Compliance') {
            agent {
                label 'secure-agent'
            }
            steps {
                // Compliance validation
                sh 'compliance-validator --framework=DoD-SRG --level=IL4'

                // STIG verification
                sh 'stig-checker'

                // Evaluate results
                script {
                    def complianceResults = readFile('compliance-results.json')
                    if (complianceResults.contains('"status": "FAIL"')) {
                        error "Compliance requirements not met"
                    }

                    // Generate compliance attestation
                    sh 'generate-compliance-attestation --framework=DoD-SRG --level=IL4'
                }
            }
            post {
                always {
                    // Archive compliance results
                    archiveArtifacts artifacts: 'compliance-results/**/*'
                }
            }
        }

        stage('Security Sign-off') {
            agent {
                label 'secure-agent'
            }
            steps {
                // Security review and approval
                timeout(time: 24, unit: 'HOURS') {
                    input message: 'Approve security review?', submitter: 'security-team'
                }
            }
        }

        stage('Production Deployment') {
            agent {
                label 'secure-agent-deploy'
            }
            steps {
                // Blue-green deployment
                sh 'blue-green-deployer --version=${VERSION}'

                // Verify deployment
                sh 'deployment-health-check --environment=production'

                // Apply runtime protection
                sh 'runtime-security-enforcer --environment=production'

                // Generate deployment attestation
                sh 'generate-deployment-attestation --version=${VERSION} --environment=production'
            }
        }

        stage('Continuous Monitoring') {
            agent {
                label 'secure-agent'
            }
            steps {
                // Enable runtime monitoring
                sh 'enable-runtime-monitoring --application=${APPLICATION_NAME} --version=${VERSION}'

                // Set up logging
                sh 'configure-centralized-logging --application=${APPLICATION_NAME}'

                // Configure alerting
                sh 'setup-security-alerting --application=${APPLICATION_NAME}'

                // Generate monitoring attestation
                sh 'generate-monitoring-attestation --application=${APPLICATION_NAME}'
            }
        }
    }

    post {
        always {
            node('secure-agent') {
                // Generate comprehensive security report
                sh 'generate-security-report --build=${BUILD_NUMBER} --application=${APPLICATION_NAME} --version=${VERSION}'

                // Archive security artifacts
                archiveArtifacts artifacts: 'security-results/**/*', fingerprint: true

                // Send security report
                emailext body: 'Security scan results attached',
                         subject: 'Pipeline Security Results - ${APPLICATION_NAME} v${VERSION}',
                         to: 'security-team@example.com',
                         attachments: 'security-results/**/*'

                // Update security dashboard
                sh 'update-security-dashboard --application=${APPLICATION_NAME} --version=${VERSION}'

                // Clean up sensitive data
                sh 'secure-workspace-cleanup'
            }
        }
        success {
            node('secure-agent') {
                // Record successful deployment
                sh 'record-deployment-success --application=${APPLICATION_NAME} --version=${VERSION}'

                // Update deployment registry
                sh 'update-deployment-registry --application=${APPLICATION_NAME} --version=${VERSION} --status=SUCCESS'
            }
        }
        failure {
            node('secure-agent') {
                // Record deployment failure
                sh 'record-deployment-failure --application=${APPLICATION_NAME} --version=${VERSION}'

                // Update deployment registry
                sh 'update-deployment-registry --application=${APPLICATION_NAME} --version=${VERSION} --status=FAILURE'

                // Send failure notification
                emailext body: 'Pipeline failed - see attached report',
                         subject: 'FAILURE: Pipeline for ${APPLICATION_NAME} v${VERSION}',
                         to: 'security-team@example.com,development-team@example.com',
                         attachments: 'security-results/**/*'
            }
        }
    }
}
```

## 4. Stage Implementations

### 4.1 Review Verification Stage

The review verification stage ensures that all required reviews and approvals have been completed before proceeding:

```groovy
stage('Review Verification') {
    agent {
        label 'secure-agent'
    }
    steps {
        // Verify peer review
        script {
            def peerReviewStatus = sh(script: 'code-review-status-checker --level=peer', returnStdout: true).trim()
            if (peerReviewStatus != 'APPROVED') {
                error "Peer review has not been completed or approved"
            }

            // Record peer review metadata
            sh 'record-review-metadata --level=peer --pipeline-run=${BUILD_NUMBER}'
        }

        // Verify security gatekeeper review
        script {
            def gatekeeperReviewStatus = sh(script: 'code-review-status-checker --level=security-gatekeeper', returnStdout: true).trim()
            if (gatekeeperReviewStatus != 'APPROVED') {
                error "Security Gatekeeper review has not been completed or approved"
            }

            // Record gatekeeper review metadata
            sh 'record-review-metadata --level=security-gatekeeper --pipeline-run=${BUILD_NUMBER}'
        }

        // Verify Mission Owner approval
        script {
            def moApprovalStatus = sh(script: 'code-review-status-checker --level=mission-owner', returnStdout: true).trim()
            if (moApprovalStatus != 'APPROVED') {
                error "Mission Owner approval has not been granted"
            }

            // Record MO approval metadata
            sh 'record-review-metadata --level=mission-owner --pipeline-run=${BUILD_NUMBER}'

            // Generate approval attestation
            sh 'generate-approval-attestation --signers=${APPROVERS} --commit=${GIT_COMMIT}'
        }
    }
}
```

### 4.2 Security Analysis Stage

The security analysis stage runs various security scanning tools in parallel:

```groovy
stage('Security Analysis') {
    parallel {
        stage('SAST') {
            agent {
                label 'secure-agent-sast'
            }
            steps {
                // Static Application Security Testing
                sh 'sonarqube-scanner'
                sh 'fortify-sca'

                // Software Composition Analysis
                sh 'dependency-check'

                // Evaluate results
                script {
                    def sastResults = readFile('sast-results.json')
                    if (sastResults.contains('"severity": "CRITICAL"')) {
                        error "Critical security vulnerabilities found in SAST scan"
                    }
                    if (sastResults.contains('"severity": "HIGH"')) {
                        error "High security vulnerabilities found in SAST scan"
                    }

                    // Generate SAST attestation
                    sh 'generate-sast-attestation --scan-id=${SAST_SCAN_ID}'
                }
            }
            post {
                always {
                    // Archive SAST results
                    archiveArtifacts artifacts: 'sast-results/**/*'
                }
            }
        }

        // ... other parallel security analysis stages ...
    }
}
```

### 4.3 Container Security Stage

The container security stage scans container images for vulnerabilities:

```groovy
stage('Container Security') {
    agent {
        label 'secure-agent'
    }
    steps {
        // Container image scanning
        sh 'trivy image ${APPLICATION_NAME}:${VERSION}'
        sh 'clair-scanner ${APPLICATION_NAME}:${VERSION}'

        // Image hardening verification
        sh 'container-compliance-scanner ${APPLICATION_NAME}:${VERSION}'

        // Evaluate results
        script {
            def containerScanResults = readFile('container-scan-results.json')
            if (containerScanResults.contains('"severity": "CRITICAL"')) {
                error "Critical vulnerabilities found in container"
            }
            if (containerScanResults.contains('"severity": "HIGH"')) {
                error "High vulnerabilities found in container"
            }

            // Generate container security attestation
            sh 'generate-container-attestation --image=${APPLICATION_NAME}:${VERSION}'
        }
    }
    post {
        always {
            // Archive container scan results
            archiveArtifacts artifacts: 'container-scan-results/**/*'
        }
    }
}
```

### 4.4 Production Deployment Stage

The production deployment stage implements blue-green deployment for safe rollouts:

```groovy
stage('Production Deployment') {
    agent {
        label 'secure-agent-deploy'
    }
    steps {
        // Blue-green deployment
        sh 'blue-green-deployer --version=${VERSION}'

        // Verify deployment
        sh 'deployment-health-check --environment=production'

        // Apply runtime protection
        sh 'runtime-security-enforcer --environment=production'

        // Generate deployment attestation
        sh 'generate-deployment-attestation --version=${VERSION} --environment=production'
    }
}
```

## 5. Tool Configurations

### 5.1 SAST Tool Configuration

Example configuration for SonarQube:

```properties
# SonarQube configuration
sonar.projectKey=${APPLICATION_NAME}
sonar.projectName=${APPLICATION_NAME}
sonar.projectVersion=${VERSION}
sonar.sources=src
sonar.tests=test
sonar.java.binaries=target/classes
sonar.java.test.binaries=target/test-classes
sonar.java.libraries=target/dependency/*.jar
sonar.coverage.jacoco.xmlReportPaths=target/site/jacoco/jacoco.xml
sonar.junit.reportPaths=target/surefire-reports
sonar.qualitygate.wait=true
```

### 5.2 Container Scanning Configuration

Example configuration for Trivy:

```yaml
# Trivy configuration
severity: "CRITICAL,HIGH"
exitCode: 1
ignoreUnfixed: true
format: "json"
output: "trivy-results.json"
timeout: "10m"
```

### 5.3 Compliance Validation Configuration

Example configuration for compliance validator:

```yaml
# Compliance validator configuration
compliance:
  frameworks:
    - name: "DoD-SRG"
      version: "1.3"
      level: "IL4"
    - name: "NIST-800-53"
      version: "Rev5"
    - name: "FedRAMP"
      version: "Moderate"
  controls:
    - id: "AC-1"
      evidence:
        - type: "document"
          path: "compliance/AC-1-policy.pdf"
    - id: "AC-2"
      evidence:
        - type: "artifact"
          path: "compliance/user-access-controls.json"
    - id: "CM-8"
      evidence:
        - type: "scan"
          path: "compliance/inventory-scan-results.json"
  reporting:
    format: "json"
    outputPath: "compliance-results/compliance-report.json"
    evidenceCollection: true
```

## 6. Deployment Considerations

### 6.1 Blue-Green Deployment Strategy

The blue-green deployment strategy ensures zero-downtime and safe rollouts:

1. **Deployment Process**

   ```bash
   #!/bin/bash
   # Blue-Green Deployment Script

   # Configuration
   APP_NAME=${APPLICATION_NAME}
   VERSION=${VERSION}
   DEPLOY_TIMESTAMP=$(date +%Y%m%d%H%M%S)

   # Determine current active environment (blue or green)
   CURRENT_ENV=$(kubectl get service ${APP_NAME}-service -o jsonpath='{.spec.selector.environment}')

   # Determine target environment
   if [ "$CURRENT_ENV" == "blue" ]; then
     TARGET_ENV="green"
   else
     TARGET_ENV="blue"
   fi

   echo "Current environment: $CURRENT_ENV"
   echo "Target environment: $TARGET_ENV"

   # Deploy to target environment
   echo "Deploying version $VERSION to $TARGET_ENV environment"
   kubectl apply -f kubernetes/${TARGET_ENV}-deployment.yaml

   # Wait for deployment to complete
   kubectl rollout status deployment/${APP_NAME}-${TARGET_ENV}

   # Run security validation in new environment
   echo "Running security validation in $TARGET_ENV environment"
   security-validation-runner --environment=${TARGET_ENV}

   # Run smoke tests
   echo "Running smoke tests in $TARGET_ENV environment"
   smoke-test-runner --environment=${TARGET_ENV}

   # Switch traffic to new environment
   echo "Switching traffic to $TARGET_ENV environment"
   kubectl patch service ${APP_NAME}-service -p '{"spec":{"selector":{"environment":"'${TARGET_ENV}'"}}}'

   # Verify traffic switch
   echo "Verifying traffic switch"
   service-traffic-verifier --service=${APP_NAME}-service --target-env=${TARGET_ENV}

   # Keep previous environment running for rollback if needed
   echo "Deployment completed successfully"
   echo "Previous environment ($CURRENT_ENV) remains available for rollback"
   ```

2. **Rollback Procedure**

   ```bash
   #!/bin/bash
   # Blue-Green Rollback Script

   # Configuration
   APP_NAME=${APPLICATION_NAME}

   # Determine current active environment (blue or green)
   CURRENT_ENV=$(kubectl get service ${APP_NAME}-service -o jsonpath='{.spec.selector.environment}')

   # Determine previous environment
   if [ "$CURRENT_ENV" == "blue" ]; then
     PREVIOUS_ENV="green"
   else
     PREVIOUS_ENV="blue"
   fi

   echo "Current environment: $CURRENT_ENV"
   echo "Rolling back to previous environment: $PREVIOUS_ENV"

   # Verify previous environment is still available
   kubectl get deployment/${APP_NAME}-${PREVIOUS_ENV} || (echo "Previous environment not available" && exit 1)

   # Switch traffic back to previous environment
   echo "Switching traffic to $PREVIOUS_ENV environment"
   kubectl patch service ${APP_NAME}-service -p '{"spec":{"selector":{"environment":"'${PREVIOUS_ENV}'"}}}'

   # Verify traffic switch
   echo "Verifying traffic switch"
   service-traffic-verifier --service=${APP_NAME}-service --target-env=${PREVIOUS_ENV}

   echo "Rollback completed successfully"
   ```

### 6.2 Runtime Protection Configuration

Runtime protection provides defense against attacks and anomalous behavior:

```yaml
# Runtime Protection Configuration
apiVersion: security.example.com/v1
kind: RuntimeProtection
metadata:
  name: ${APPLICATION_NAME}-protection
  namespace: production
spec:
  target:
    application: ${APPLICATION_NAME}
    version: ${VERSION}
    selector:
      matchLabels:
        app: ${APPLICATION_NAME}
  protections:
    fileSystem:
      readOnlyRootFilesystem: true
      allowedWritePaths:
        - /tmp
        - /var/log
    network:
      egressControl: true
      allowedDestinations:
        - domain: api.example.com
          ports: [443]
        - domain: database.example.com
          ports: [5432]
    processes:
      restrictExecution: true
      allowedExecutables:
        - /usr/bin/java
        - /usr/bin/node
    behavior:
      anomalyDetection: true
      baselineTrainingPeriod: 7d
    vulnerabilityExploitation:
      preventExploitation: true
      cveProtection: true
  response:
    alert:
      enabled: true
      severity: high
    block:
      enabled: true
      logOnly: false
    forensics:
      enabled: true
      evidenceCollection: true
  logging:
    centralizedLogging: true
    logLevel: INFO
    sensitiveDataMasking: true
```

### 6.3 Security Monitoring Integration

Security monitoring ensures continuous visibility into application security:

```yaml
# Security Monitoring Configuration
apiVersion: monitoring.example.com/v1
kind: SecurityMonitoring
metadata:
  name: ${APPLICATION_NAME}-monitoring
  namespace: production
spec:
  application: ${APPLICATION_NAME}
  version: ${VERSION}
  components:
    - logs:
        enabled: true
        sources:
          - container: app
            path: /var/log/app.log
          - container: sidecar
            path: /var/log/security.log
        parsers:
          - type: json
            fields:
              - timestamp
              - level
              - message
              - user
    - metrics:
        enabled: true
        endpoints:
          - port: 9090
            path: /metrics
        collectors:
          - type: prometheus
            interval: 15s
    - traces:
        enabled: true
        samplingRate: 0.1
        collectors:
          - type: jaeger
            endpoint: jaeger-collector:14250
  alerting:
    rules:
      - name: high-error-rate
        condition: "sum(rate(app_http_errors[5m])) / sum(rate(app_http_requests[5m])) > 0.05"
        severity: warning
      - name: security-violation
        condition: "sum(security_violations) > 0"
        severity: critical
    destinations:
      - type: security-team
        channel: security-alerts
      - type: incident-response
        channel: security-incidents
  dashboards:
    - name: security-overview
      provider: grafana
      path: /dashboards/security-overview.json
    - name: vulnerability-tracking
      provider: grafana
      path: /dashboards/vulnerability-tracking.json
```

## 7. Pipeline Attestations

Each stage of the pipeline produces security attestations for validation and verification:

### 7.1 Attestation Format

```json
{
  "attestation": {
    "type": "security-attestation",
    "version": "1.0",
    "metadata": {
      "applicationName": "${APPLICATION_NAME}",
      "applicationVersion": "${VERSION}",
      "buildId": "${BUILD_NUMBER}",
      "timestamp": "${BUILD_TIMESTAMP}",
      "attestationId": "${ATTESTATION_ID}"
    },
    "stage": {
      "name": "container-security",
      "status": "passed",
      "timestamp": "${STAGE_TIMESTAMP}"
    },
    "evidence": {
      "scanId": "${SCAN_ID}",
      "scannerName": "trivy",
      "scannerVersion": "0.38.0",
      "findings": {
        "critical": 0,
        "high": 0,
        "medium": 3,
        "low": 12,
        "informational": 5
      },
      "thresholds": {
        "critical": 0,
        "high": 0,
        "medium": 5
      },
      "resultsLocation": "container-scan-results/trivy-results.json"
    },
    "approvals": [
      {
        "approverRole": "security-scanner",
        "approverName": "automated-scanner",
        "timestamp": "${APPROVAL_TIMESTAMP}",
        "signature": "${APPROVAL_SIGNATURE}"
      }
    ],
    "signature": {
      "algorithm": "SHA256withRSA",
      "value": "${ATTESTATION_SIGNATURE}",
      "keyId": "${SIGNING_KEY_ID}"
    }
  }
}
```

### 7.2 Attestation Chain

The pipeline produces a chain of attestations covering each security aspect:

1. **Source Attestation**: Verifies code source and integrity
2. **Review Attestation**: Captures the multi-tier review process
3. **SAST Attestation**: Documents static analysis results
4. **SCA Attestation**: Documents dependency analysis
5. **Container Attestation**: Verifies container security
6. **IaC Attestation**: Documents infrastructure security
7. **Compliance Attestation**: Verifies compliance status
8. **Deployment Attestation**: Documents secure deployment
9. **Monitoring Attestation**: Confirms security monitoring

### 7.3 Attestation Verification

```bash
#!/bin/bash
# Attestation Verification Script

# Configuration
APP_NAME=${APPLICATION_NAME}
VERSION=${VERSION}
BUILD_ID=${BUILD_NUMBER}

# Verify attestation chain
echo "Verifying attestation chain for $APP_NAME v$VERSION (Build $BUILD_ID)"

# Get attestation locations
ATTESTATION_DIR="attestations/${APP_NAME}/${VERSION}"

# Verify each attestation
for attestation_file in $(find $ATTESTATION_DIR -name "*.json"); do
  echo "Verifying attestation: $attestation_file"

  # Verify signature
  cosign verify-attestation $attestation_file || exit 1

  # Check attestation status
  status=$(jq -r '.attestation.stage.status' $attestation_file)
  if [ "$status" != "passed" ]; then
    echo "Attestation failed: $attestation_file"
    exit 1
  fi

  echo "Attestation verified: $attestation_file"
done

echo "All attestations verified successfully"
```

## 8. Conclusion

This implementation example demonstrates a comprehensive CI/CD pipeline that incorporates all the security controls required by the DoD Cloud Security Playbook. By following this example, organizations can establish a secure pipeline that:

1. Enforces multi-tier code review with proper approvals
2. Implements comprehensive security testing
3. Ensures infrastructure and container security
4. Provides continuous compliance validation
5. Enables secure deployment practices
6. Establishes runtime protection and monitoring

Each organization should adapt this example to their specific needs and technologies while maintaining the core security principles and controls.

[← Back to Main Document](README.md)
