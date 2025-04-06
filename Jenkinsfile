import io.jenkins.plugins.checks.api.ChecksStatus
def globalServiceChanged = []

pipeline {
    agent any

    stages {
        stage('Checkout') {
            steps {
                script {
                    def changes = sh(script: 'git diff --name-only HEAD~1 HEAD', returnStdout: true).trim()
                    echo "Changed files:\n${changes}"

                    def serviceList = [
                        "spring-petclinic-admin-server",
                        "spring-petclinic-api-gateway",
                        "spring-petclinic-config-server",
                        "spring-petclinic-customers-service",
                        "spring-petclinic-discovery-server",
                        "spring-petclinic-genai-service",
                        "spring-petclinic-vets-service",
                        "spring-petclinic-visits-service"
                    ]

                    for (svc in serviceList) {
                        if (changes.contains("${svc}/")) {
                            globalServiceChanged << svc
                        }
                    }

                    if (globalServiceChanged.isEmpty()) {
                        echo "No relevant changes detected, skipping build."
                        currentBuild.result = 'ABORTED'
                        return
                    }

                    echo "Changed services: ${globalServiceChanged}"
                }
            }
        }

        stage('Test') {
            when {
                expression { globalServiceChanged && globalServiceChanged.size() > 0 }
            }
            steps {
                script {
                    globalServiceChanged.each { svc ->
                        dir("${svc}") {
                            sh '../mvnw test'
                        }
                    }
                }
            }
            post {
                always {
                    script {
                        globalServiceChanged.each { svc ->
                            def reportPath = "${svc}/target/surefire-reports"
                            if (fileExists(reportPath)) {
                                junit "${svc}/target/surefire-reports/*.xml"
                                cobertura coberturaReportFile: "${svc}/target/site/cobertura/coverage.xml"
                            } else {
                                echo "No test reports found for ${svc}."
                            }
                        }
                    }
                }
            }
        }

        stage('Coverage Check') {
            when {
                expression { globalServiceChanged && globalServiceChanged.size() > 0 }
            }
            steps {
                script {
                    globalServiceChanged.each { svc ->
                        def coverageFile = "${svc}/target/site/jacoco/index.html"
                        if (fileExists(coverageFile)) {
                            def coverage = sh(script: "grep -oP '(?<=coverage: )\\d+' ${coverageFile} | head -1", returnStdout: true).trim()
                            if (coverage.toInteger() < 70) {
                                error "❌ ${svc}: Coverage is ${coverage}%, below required threshold (70%)"
                            }
                            echo "✅ ${svc}: Coverage is ${coverage}% - OK!"
                        } else {
                            echo "⚠️ ${svc}: Coverage report not found."
                        }
                    }
                }
            }
        }

        stage('Build') {
            when {
                expression { globalServiceChanged && globalServiceChanged.size() > 0 }
            }
            steps {
                script {
                    globalServiceChanged.each { svc ->
                        dir("${svc}") {
                            sh '../mvnw package'
                        }
                    }
                }
            }
        }
    }

    // Có thể bật lại nếu bạn dùng GitHub Checks
    // post {
    //     success {
    //         publishChecks name: 'Jenkins', status: ChecksStatus.COMPLETED
    //     }
    //     failure {
    //         publishChecks name: 'Jenkins', status: ChecksStatus.COMPLETED
    //     }
    // }
}
