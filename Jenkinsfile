// FULL PIPELINE (Pipeline-from-SCM): checkout -> build -> image -> deploy
pipeline {
  agent any
  tools { maven 'Maven3'; jdk 'JDK17' }
  environment {
    DOCKERHUB_USER = 'YOUR_DOCKERHUB_USER'
    SERVICES = 'product-service order-service user-service'
  }
  stages {
    stage('Checkout') {
      steps {
        checkout scm
        script { env.IMAGE_TAG = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim() }
      }
    }
    stage('Build & Test') {
      steps { sh 'mvn -B clean verify' }
    }
    stage('Docker Images') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'dockerhub-creds',
                         usernameVariable: 'DH_USER', passwordVariable: 'DH_PASS')]) {
          sh '''
            echo "$DH_PASS" | docker login -u "$DH_USER" --password-stdin
            for svc in $SERVICES; do
              docker build -t $DOCKERHUB_USER/$svc:$IMAGE_TAG -t $DOCKERHUB_USER/$svc:latest $svc
              docker push $DOCKERHUB_USER/$svc:$IMAGE_TAG
              docker push $DOCKERHUB_USER/$svc:latest
            done
            docker logout
          '''
        }
      }
    }
    stage('Deploy') {
      steps {
        withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG')]) {
          sh '''
            for svc in $SERVICES; do
              sed -e "s|DOCKERHUB_USER|$DOCKERHUB_USER|g" -e "s|IMAGE_TAG|$IMAGE_TAG|g" $svc/k8s/deployment.yaml | kubectl apply -f -
              kubectl apply -f $svc/k8s/service.yaml
              kubectl rollout status deployment/$svc --timeout=180s
            done
          '''
        }
      }
    }
  }
}
