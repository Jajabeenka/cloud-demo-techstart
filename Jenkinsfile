pipeline {
  agent any

  environment {
    AZ_ACCOUNT = 'carranzastg2'
    AZ_SHARE   = 'webcontent'
  }

  stages {

    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    stage('Deploy to ACI (file share)') {
      steps {
        withCredentials([string(credentialsId: 'azure-storage-key', variable: 'AZ_KEY')]) {

          sh '''
            pwd
            ls -la
            find . -name "*.html"

            az storage file upload-batch \
  --account-name carranzastg2 \
  --account-key "$AZ_KEY" \
  --destination webcontent \
  --source . \
  --no-progress
          '''
        }
      }
    }
  }
}