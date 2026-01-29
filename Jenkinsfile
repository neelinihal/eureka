pipeline {
  agent any

  options {
    timestamps()
    disableConcurrentBuilds()
  }

  environment {
    REPO_URL     = 'https://github.com/neelinihal/eureka'
    GIT_BRANCH   = 'myCodes'
    MODULE_DIR   = '.'
    IMAGE_NAME   = 'neelinihal/eureka'
    IMAGE_TAG    = "build-${BUILD_NUMBER}"
    DOCKER_CREDS = 'dockerhub-creds'
    KUBE_NS      = 'default'
    DEPLOY_NAME  = 'eureka'
    GCP_CREDS    = 'gcp-sa-json'
    CLUSTER_NAME = 'my-cluster'
    CLUSTER_ZONE = 'us-central1'
    PROJECT_ID   = 'steel-earth-478506-t2'
  }

  stages {

    stage('Checkout') {
      steps {
        git branch: "${GIT_BRANCH}", url: "${REPO_URL}"
      }
    }

    stage('Build (Maven)') {
      steps {
        dir(MODULE_DIR) {
          sh '''
            java -version
            mvn -version
            mvn clean package
          '''
        }
      }
    }

    stage('Docker Build') {
      steps {
        dir(MODULE_DIR) {
          sh "docker build -t ${IMAGE_NAME}:${IMAGE_TAG} ."
        }
      }
    }

    stage('Docker Login & Push') {
      steps {
        withCredentials([usernamePassword(
          credentialsId: DOCKER_CREDS,
          usernameVariable: 'DOCKER_USER',
          passwordVariable: 'DOCKER_PASS'
        )]) {
          sh '''
            echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
            docker push ${IMAGE_NAME}:${IMAGE_TAG}
          '''
        }
      }
    }

    stage('Update Deployment YAML') {
      steps {
        sh '''
          sed -i "s|image: neelinihal/eureka:.*|image: neelinihal/eureka:${IMAGE_TAG}|g" deployment.yaml
        '''
      }
    }

    stage('Authenticate GCP') {
      steps {
        withCredentials([file(credentialsId: GCP_CREDS, variable: 'GOOGLE_APPLICATION_CREDENTIALS')]) {
          sh '''
            gcloud auth activate-service-account --key-file=$GOOGLE_APPLICATION_CREDENTIALS
            gcloud config set project ${PROJECT_ID}
            gcloud container clusters get-credentials ${CLUSTER_NAME} \
              --zone ${CLUSTER_ZONE} \
              --project ${PROJECT_ID}
          '''
        }
      }
    }

    stage('Deploy to GKE') {
      steps {
        sh '''
          kubectl apply -f deployment.yaml -n ${KUBE_NS}
          kubectl rollout status deployment/${DEPLOY_NAME} -n ${KUBE_NS}
        '''
      }
    }
  }
}
