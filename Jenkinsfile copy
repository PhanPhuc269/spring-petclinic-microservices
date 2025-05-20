
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
    stage('Build & Push Changed Services') {
      steps {
        script {
          def services = [
            [name: 'config-server', branch: params.config_branch],
            [name: 'discovery-server', branch: params.discovery_branch],
            [name: 'customers-service', branch: params.customers_branch],
            [name: 'vets-service', branch: params.vets_branch],
            [name: 'visits-service', branch: params.visits_branch],
            [name: 'api-gateway', branch: params.gateway_branch],
            [name: 'admin-server', branch: params.admin_branch]
          ]

          def imageTags = [:] // store service => image tag

          for (svc in services) {
            def name = svc.name
            def branch = svc.branch
            def tag = 'latest'

            if (branch != 'main') {
              dir("${name}") {
                echo "➡️ Building ${name} from branch ${branch}"
                sh "git checkout ${branch}"
                dir("spring-petclinic-${name}") {
                  sh "../../../mvnw clean install -P buildDocker -DskipTests"
                  def commitId = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
                  tag = commitId
                  withCredentials([usernamePassword(credentialsId: 'dockerhub', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                        sh "echo \$DOCKER_PASS | docker login -u \$DOCKER_USER --password-stdin"
                        sh "docker tag springcommunity/spring-petclinic-${name}:latest ${DOCKERHUB_REPO}/spring-petclinic-${name}:${tag}"
                        sh "docker push ${DOCKERHUB_REPO}/spring-petclinic-${name}:${tag}"
                    }

                }
              }
            }
            imageTags[name] = tag
          }

          // Save tags to environment for Helm deploy
          writeFile file: 'imagetags.properties', text: imageTags.collect { k, v -> "${k}=${v}" }.join('\n')
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