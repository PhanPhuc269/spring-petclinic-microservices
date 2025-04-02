pipeline {
    agent any
    environment {
        SERVICE_CHANGED = ""  // Biến để kiểm tra service nào thay đổi
    }
    stages {
        stage('Checkout') {
            steps {
                script {
                    // Lấy danh sách file đã thay đổi
                    def changes = sh(script: 'git diff --name-only HEAD~1 HEAD', returnStdout: true).trim()
                    echo "Changed files:\n${changes}"
                    
                    // Xác định service nào cần build
                    if (changes.contains("customers-service/")) {
                        env.SERVICE_CHANGED = "customers-service"
                    } else if (changes.contains("vets-service/")) {
                        env.SERVICE_CHANGED = "vets-service"
                    } else if (changes.contains("visit-service/")) {
                        env.SERVICE_CHANGED = "visit-service"
                    } else {
                        echo "No relevant changes detected, skipping build."
                        currentBuild.result = 'ABORTED'
                        return
                    }
                    echo "Service to build: ${env.SERVICE_CHANGED}"
                }
            }
        }
        
        stage('Test') {
            when { expression { env.SERVICE_CHANGED != "" } }
            steps {
                dir("${env.SERVICE_CHANGED}") {
                    sh './mvnw test' // Chạy test
                }
            }
            post {
                success {
                    echo "Tests passed successfully."
                }
                failure {
                    error "Tests failed!"
                }
            }
        }
        
        stage('Coverage Check') {
            when { expression { env.SERVICE_CHANGED != "" } }
            steps {
                script {
                    def coverage = sh(script: "grep -oP '(?<=coverage: )\\d+' ${env.SERVICE_CHANGED}/target/site/jacoco/index.html | head -1", returnStdout: true).trim()
                    if (coverage.toInteger() < 70) {
                        error "Coverage is ${coverage}%, below required threshold (70%)"
                    }
                    echo "Coverage is ${coverage}% - OK!"
                }
            }
        }
        
        stage('Build') {
            when { expression { env.SERVICE_CHANGED != "" } }
            steps {
                dir("${env.SERVICE_CHANGED}") {
                    sh './mvnw package' // Build service
                }
            }
        }
    }
    
    post {
        always {
            junit '**/target/surefire-reports/*.xml' // Upload test results
        }
    }
}
