def globalServiceChanged = []
def commitId = ''
def isTagBuild = false
def gitTagName = ''

pipeline {
    agent any
    environment {
        DOCKERHUB_REPO = 'phanphuc269/' 
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
                        echo "**Tag build detected: ${gitTagName}"

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

        stage('Clone helm-petclinic') {
            steps {
                script {
                    dir('helm') {
                        git branch: 'main', url: 'https://github.com/PhanPhuc269/helm-petclinic.git'
                        sh 'git config user.email "jenkins@yourdomain.com"'
                        sh 'git config user.name "Jenkins CI"'
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
                                    echo "#Building image: ${imageTag}"
                                    sh 'export DOCKER_BUILDKIT=1 && ../mvnw clean install -P buildDocker -DskipTests'

                                    sh "docker tag springcommunity/${svc}:latest ${imageTag}"
                                    echo "#Pushing image: ${imageTag}"
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
                        sh "
                            sed -i '/${key}:\\\$/,/tag:/s/tag: .*/tag: ${commitId}/' ./helm/environments/values-dev.yaml
                        "
                    }
                    withCredentials([usernamePassword(credentialsId: 'PhanPhuc269', usernameVariable: 'GIT_USER', passwordVariable: 'GIT_TOKEN')]) {
                        dir('helm') {
                            sh 'git config user.email "jenkins@yourdomain.com"'
                            sh 'git config user.name "Jenkins CI"'
                            sh 'git add environments/values-dev.yaml'
                            sh "git commit -m '[dev] Update image tag to ${commitId} for ${globalServiceChanged}' || true"
                            def remoteUrl = "https://${GIT_USER}:${GIT_TOKEN}@github.com/PhanPhuc269/helm-petclinic.git"
                            sh "git remote set-url origin ${remoteUrl}"
                            sh 'git push origin HEAD:dev'
                        }
                    }
                }
            }
        }

        stage('Update values-staging.yaml & Push to Git (staging)') {
            when {
                expression { isTagBuild }
            }
            steps {
                script {
                    sh "sed -i 's/tag: .*/tag: ${gitTagName}/g' ./helm/environments/values-staging.yaml"
                    
                    withCredentials([usernamePassword(credentialsId: 'PhanPhuc269', usernameVariable: 'GIT_USER', passwordVariable: 'GIT_TOKEN')]) {
                        dir('helm') {
                            sh 'git config user.email "jenkins@yourdomain.com"'
                            sh 'git config user.name "Jenkins CI"'
                            sh 'git add environments/values-staging.yaml'
                            sh "git commit -m '[staging] Release image tag ${gitTagName} for all services' || true"
                            def remoteUrl = "https://${GIT_USER}:${GIT_TOKEN}@github.com/PhanPhuc269/helm-petclinic.git"
                            sh "git remote set-url origin ${remoteUrl}"
                            sh 'git push origin HEAD:staging'
                        }
                    }
                }
            }
        } 
    }

    post {
        success {
            script {
                if (isTagBuild) {
                    echo "#Success: Successfully built and pushed all services for tag: ${gitTagName}"
                } else {
                    echo "#Sucess: Build and push completed for changed services: ${globalServiceChanged}"
                }
            }
        }
    }
}
