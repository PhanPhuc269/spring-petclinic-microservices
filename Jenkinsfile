// Jenkinsfile tổng quát cho CI/CD (Petclinic)

def commitId = ""

def SERVICE_NAME = params.SERVICE_NAME

def NAMESPACE = params.NAMESPACE

def DOMAIN = params.DOMAIN

def BRANCH = params.BRANCH_NAME

def DOCKER_IMAGE = "hykura/spring-petclinic-${SERVICE_NAME}"

def RELEASE_NAME = "${SERVICE_NAME}-${NAMESPACE}"

def IMAGE_TAG = ""

pipeline {
  agent any

  parameters {
    string(name: 'BRANCH_NAME', defaultValue: 'main', description: 'Git branch')
    string(name: 'SERVICE_NAME', defaultValue: 'vets-service', description: 'Tên service cần test')
    string(name: 'NAMESPACE', defaultValue: 'dev-a', description: 'Namespace riêng cho dev')
    string(name: 'DOMAIN', defaultValue: 'dev-a.local', description: 'Domain dev truy cập')
  }

  environment {
    DOCKERHUB_REPO = 'hykura'
  }

  stages {

    stage('Clone') {
      steps {
        git branch: "${BRANCH}", url: 'https://github.com/spring-petclinic/spring-petclinic-microservices.git'
      }
    }

    stage('Get Commit ID') {
      steps {
        script {
          commitId = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
          IMAGE_TAG = commitId
          echo "✅ Commit ID: ${commitId}"
        }
      }
    }

    stage('Build & Push Image') {
      steps {
        dir("spring-petclinic-${SERVICE_NAME}") {
          script {
            sh "../mvnw clean install -P buildDocker -DskipTests"
            sh "docker tag springcommunity/spring-petclinic-${SERVICE_NAME}:latest ${DOCKERHUB_REPO}/spring-petclinic-${SERVICE_NAME}:${IMAGE_TAG}"
            sh "docker push ${DOCKERHUB_REPO}/spring-petclinic-${SERVICE_NAME}:${IMAGE_TAG}"
          }
        }
      }
    }

    stage('Deploy with Helm') {
      steps {
        script {
          sh "kubectl create namespace ${NAMESPACE} || true"

          sh """
            helm upgrade --install ${RELEASE_NAME} ./petclinic-chart \
              --namespace ${NAMESPACE} \
              --create-namespace \
              --set image.repository=${DOCKERHUB_REPO}/spring-petclinic-${SERVICE_NAME} \
              --set image.tag=${IMAGE_TAG} \
              --set ingress.host=${DOMAIN}
          """
        }
      }
    }
  }

  post {
    success {
      echo "✅ Build + Deploy thành công: ${SERVICE_NAME} @ ${DOMAIN} (namespace: ${NAMESPACE})"
    }
    failure {
      echo "❌ Pipeline thất bại. Kiểm tra log để biết chi tiết."
    }
  }
}