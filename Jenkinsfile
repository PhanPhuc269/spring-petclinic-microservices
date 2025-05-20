
pipeline {
  agent any

  parameters {
    string(name: 'config_branch', defaultValue: 'main')
    string(name: 'discovery_branch', defaultValue: 'main')
    string(name: 'customers_branch', defaultValue: 'main')
    string(name: 'vets_branch', defaultValue: 'main')
    string(name: 'visits_branch', defaultValue: 'main')
    string(name: 'gateway_branch', defaultValue: 'main')
    string(name: 'admin_branch', defaultValue: 'main')
    string(name: 'NAMESPACE', defaultValue: 'dev-a')
    string(name: 'DOMAIN', defaultValue: 'dev-a.local')
  }

  environment {
    DOCKERHUB_REPO = 'phanphuc269'
  }

  stages {
        stage('Build & Push Docker Images') {
            when {
                expression { globalServiceChanged.size() > 0 }
            }
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub-cred', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    script {
                        sh "echo \$DOCKER_PASS | docker login -u \$DOCKER_USER --password-stdin"

                        def branches = [:]

                        globalServiceChanged.each { svc ->
                            branches[svc] = {
                                dir("${svc}") {
                                    def imageTag = "${DOCKERHUB_REPO}${svc}:${commitId}"
                                    echo "Building image: ${imageTag}"
                                    sh '../mvnw clean install -P buildDocker -DskipTests'
                                    sh "docker tag springcommunity/${svc}:latest ${imageTag}"
                                    sh "docker push ${imageTag}"
                                }
                            }
                        }

                        parallel branches
                    }
                }
            }
        }


    stage('Deploy with Helm') {
      steps {
        script {
          def tagMap = readProperties file: 'imagetags.properties'

          sh "kubectl create namespace ${params.NAMESPACE} || true"

          def helmArgs = [
            "--namespace ${params.NAMESPACE}",
            "--create-namespace",
            "--set ingress.host=${params.DOMAIN}"
          ]

          for (entry in tagMap) {
            def svc = entry.key
            def tag = entry.value
            helmArgs << "--set ${svc.replace('-', '').replace('service', '')}.image.tag=${tag}"
            helmArgs << "--set ${svc.replace('-', '').replace('service', '')}.image.repository=${DOCKERHUB_REPO}/spring-petclinic-${svc}"
          }

          sh "helm upgrade --install petclinic ./petclinic-chart ${helmArgs.join(' ')}"
        }
      }
    }
  }

  post {
    success {
      echo "✅ CI/CD thành công. Mỗi service được gắn tag theo branch chỉ định."
    }
    failure {
      echo "❌ Pipeline thất bại. Kiểm tra lại thông số hoặc log chi tiết."
    }
  }
}