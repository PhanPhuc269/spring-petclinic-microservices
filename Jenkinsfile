import io.jenkins.plugins.checks.api.ChecksStatus

pipeline {
    agent any
    environment {
        SERVICE_CHANGED = "" // Biến kiểm tra service thay đổi
    }
    stages {
        stage('Checkout') {
            steps {
                script {
                    // Lấy danh sách file đã thay đổi
                    def changes = sh(script: 'git diff --name-only HEAD~1 HEAD', returnStdout: true).trim()
                    echo "Changed files:\n${changes}"

                    // Xác định service nào thay đổi
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
        
        // stage('Test') {
        //     when { expression { env.SERVICE_CHANGED != "" } }
        //     steps {
        //         script {
        //             // Kiểm tra nội dung thư mục trước khi chạy
        //             sh 'ls -lah'
                    
        //             // Di chuyển vào thư mục chứa source code nếu cần
        //             sh './mvnw test'
        //         }
        //     }
        //     post {
        //         success {
        //             echo "✅ Tests passed successfully."
        //         }
        //         failure {
        //             error "❌ Tests failed!"
        //         }
        //     }
        // }
        stage('Test') {
            when {
                expression { env.SERVICE_CHANGED != "" }
            }
            steps {
                script{
                    sh "cd ${env.SERVICE_CHANGED}"
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
            when { expression { env.SERVICE_CHANGED != "" } }
            steps {
                script {
                    def coverageFile = "spring-petclinic-microservices/${env.SERVICE_CHANGED}/target/site/jacoco/index.html"
                    if (fileExists(coverageFile)) {
                        def coverage = sh(script: "grep -oP '(?<=coverage: )\\d+' ${coverageFile} | head -1", returnStdout: true).trim()
                        if (coverage.toInteger() < 70) {
                            error "❌ Coverage is ${coverage}%, below required threshold (70%)"
                        }
                        echo "✅ Coverage is ${coverage}% - OK!"
                    } else {
                        echo "⚠️ Coverage report not found. Skipping coverage check."
                    }
                }
            }
        }
        
        stage('Build') {
            when { expression { env.SERVICE_CHANGED != "" } }
            steps {
                script {
                    sh "cd ${env.SERVICE_CHANGED}"
                    sh '../mvnw package'
                }
            }
        }
    }
    
    post {
        success {
            publishChecks name: 'Jenkins', status: ChecksStatus.COMPLETED
        }
        failure {
            publishChecks name: 'Jenkins', status: ChecksStatus.COMPLETED
        }
    }

}
