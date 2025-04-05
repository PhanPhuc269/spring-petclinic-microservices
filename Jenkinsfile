@Library('github-checks') _  // Chỉ cần nếu dùng publishChecks (có thể bỏ nếu không dùng)

import io.jenkins.plugins.checks.api.ChecksStatus
import io.jenkins.plugins.checks.api.ChecksConclusion

pipeline {
    agent any

    tools {
        maven 'Maven 3.8.8'
        jdk 'Java 17'
    }

    environment {
        MAVEN_OPTS = "-Dmaven.test.failure.ignore=false"
    }

    options {
        timestamps()
        skipDefaultCheckout(false)
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Detect Changed Service') {
            steps {
                script {
                    def changes = sh(script: 'git diff --name-only HEAD~1', returnStdout: true).trim()
                    echo "Changed files:\n${changes}"

                    env.SERVICE_CHANGED = ""

                    if (changes.contains("vets-service/")) {
                        env.SERVICE_CHANGED = "vets-service"
                    } else if (changes.contains("customers-service/")) {
                        env.SERVICE_CHANGED = "customers-service"
                    } else if (changes.contains("visits-service/")) {
                        env.SERVICE_CHANGED = "visits-service"
                    }

                    if (env.SERVICE_CHANGED == "") {
                        echo "Không có service nào thay đổi. Dừng pipeline."
                        currentBuild.result = 'ABORTED'
                        return
                    }

                    echo "Service bị thay đổi: ${env.SERVICE_CHANGED}"
                }
            }
        }

        stage('Test') {
            when {
                expression { env.SERVICE_CHANGED != "" }
            }
            steps {
                dir("${env.SERVICE_CHANGED}") {
                    sh '../mvnw test'
                }
            }
            post {
                always {
                    junit "${env.SERVICE_CHANGED}/target/surefire-reports/*.xml"
                    cobertura coberturaReportFile: "${env.SERVICE_CHANGED}/target/site/cobertura/coverage.xml"
                }
            }
        }

        stage('Coverage Check') {
            when {
                expression { env.SERVICE_CHANGED != "" }
            }
            steps {
                script {
                    def coverageHtml = "${env.SERVICE_CHANGED}/target/site/jacoco/index.html"
                    def coverage = sh(script: "grep -oP '(?<=coverage: )\\d+' ${coverageHtml} | head -1", returnStdout: true).trim()

                    echo "Coverage: ${coverage}%"

                    if (coverage.toInteger() < 70) {
                        error "Coverage dưới 70%. Không cho phép merge."
                    }
                }
            }
        }

        stage('Build') {
            when {
                expression { env.SERVICE_CHANGED != "" }
            }
            steps {
                dir("${env.SERVICE_CHANGED}") {
                    sh '../mvnw package'
                }
            }
        }
    }

    post {
        success {
            publishChecks name: 'Jenkins CI',
                status: ChecksStatus.COMPLETED,
                conclusion: ChecksConclusion.SUCCESS
        }
        failure {
            publishChecks name: 'Jenkins CI',
                status: ChecksStatus.COMPLETED,
                conclusion: ChecksConclusion.FAILURE
        }
        aborted {
            echo "Pipeline bị dừng (ABORTED)."
        }
    }
} 
