def globalServiceChanged = []
def commitId = ''
def isTagBuild = false
def gitTagName = ''

pipeline {
    agent any
    environment {
        DOCKERHUB_REPO = 'phanphuc269/' // Thay bằng DockerHub repo của bạn
    }

    stages {
        stage('Checkout') {
            steps {
                script {
                    checkout scm
                    commitId = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
                    gitTagName = sh(script: 'git describe --exact-match --tags || true', returnStdout: true).trim()

                    if (gitTagName.startsWith("v")) {
                        isTagBuild = true
                        echo "📦 Tag build detected: ${gitTagName}"
                        // Khi build theo tag, build toàn bộ service
                        globalServiceChanged = [
                            "spring-petclinic-admin-server",
                            "spring-petclinic-api-gateway",
                            "spring-petclinic-config-server",
                            "spring-petclinic-customers-service",
                            "spring-petclinic-discovery-server",
                            "spring-petclinic-vets-service",
                            "spring-petclinic-visits-service"
                        ]
                    } else {
                        def changes = sh(script: 'git diff --name-only HEAD~1 HEAD', returnStdout: true).trim()
                        def serviceList = [
                            "spring-petclinic-admin-server",
                            "spring-petclinic-api-gateway",
                            "spring-petclinic-config-server",
                            "spring-petclinic-customers-service",
                            "spring-petclinic-discovery-server",
                            "spring-petclinic-vets-service",
                            "spring-petclinic-visits-service"
                        ]

                        for (svc in serviceList) {
                            if (changes.contains("${svc}/")) {
                                globalServiceChanged << svc
                            }
                        }

                        echo "Changed services: ${globalServiceChanged}"
                        echo "Commit ID: ${commitId}"
                    }
                }
            }
        }

        stage('Build & Push Docker Images') {
            when {
                expression { globalServiceChanged.size() > 0 }
            }
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    script {
                        sh "echo \$DOCKER_PASS | docker login -u \$DOCKER_USER --password-stdin"
                        def branches = [:]

                        globalServiceChanged.each { svc ->
                            branches[svc] = {
                                dir("${svc}") {
                                    def tag = isTagBuild ? gitTagName : commitId
                                    def imageTag = "${DOCKERHUB_REPO}${svc}:${tag}"

                                    echo "🐳 Building image: ${imageTag}"
                                    sh 'export DOCKER_BUILDKIT=1 && ../mvnw clean install -P buildDocker -DskipTests'

                                    sh "docker tag springcommunity/${svc}:latest ${imageTag}"
                                    echo "🚀 Pushing image: ${imageTag}"
                                    sh "docker push ${imageTag}"
                                }
                            }
                        }

                        parallel branches
                    }
                }
            }
        }

        stage('Update values-dev.yaml & Push to Git (dev)') {
            when {
                expression { !isTagBuild && globalServiceChanged.size() > 0 }
            }
            steps {
                script {
                    globalServiceChanged.each { svc ->
                        def key = svc.replace("spring-petclinic-", "").replace("-service", "-service").replace("-gateway", "-gateway")
                        powershell """
                        (Get-Content d:/phuc/DevOps/DA2/helm-petclinic/environments/values-dev.yaml) `
                        -replace '(?<=${key}:\\s*\\n\\s*image:\\s*\\n\\s*tag: )\\S+', '${commitId}' `
                        | Set-Content d:/phuc/DevOps/DA2/helm-petclinic/environments/values-dev.yaml
                        """
                    }
                    powershell 'git config user.email "jenkins@yourdomain.com"'
                    powershell 'git config user.name "Jenkins CI"'
                    powershell 'git add d:/phuc/DevOps/DA2/helm-petclinic/environments/values-dev.yaml'
                    powershell "git commit -m \"[dev] Update image tag to ${commitId} for ${globalServiceChanged}\""
                    powershell 'git push origin HEAD:dev'
                }
            }
        }

        stage('Update values-staging.yaml & Push to Git (staging)') {
            when {
                expression { isTagBuild }
            }
            steps {
                script {
                    // Cập nhật tag cho tất cả service về tag mới (gitTagName)
                    powershell """
                    (Get-Content d:/phuc/DevOps/DA2/helm-petclinic/environments/values-staging.yaml) `
                    -replace '(?<=tag: )\\S+', '${gitTagName}' `
                    | Set-Content d:/phuc/DevOps/DA2/helm-petclinic/environments/values-staging.yaml
                    """
                    powershell 'git config user.email "jenkins@yourdomain.com"'
                    powershell 'git config user.name "Jenkins CI"'
                    powershell 'git add d:/phuc/DevOps/DA2/helm-petclinic/environments/values-staging.yaml'
                    powershell "git commit -m \"[staging] Release image tag ${gitTagName} for all services\""
                    powershell 'git push origin HEAD:staging'
                }
            }
        }        
    }

    post {
        success {
            script {
                if (isTagBuild) {
                    echo "✅ Successfully built and pushed all services for tag: ${gitTagName}"
                } else {
                    echo "✅ Build and push completed for changed services: ${globalServiceChanged}"
                }
            }
        }
    }
}
