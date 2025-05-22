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
