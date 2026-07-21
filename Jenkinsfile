pipeline {
 
  agent any
 
  environment {
 
    AZ_ACCOUNT = 'carranzastg2'   // same as $STG
 
    AZ_SHARE   = 'webcontent'
 
  }
 
  stages {
 
    stage('Checkout') { steps { checkout scm } }
 
    stage('Deploy to ACI (file share)') {
 
      steps {
 
        withCredentials([string(credentialsId: 'azure-storage-key', variable: 'AZ_KEY')]) {
 
          sh '''
 
            az storage file upload-batch \
 
              --account-name "$AZ_ACCOUNT" --account-key "$AZ_KEY" \
 
              --destination "$AZ_SHARE" --source . \
 
              --pattern "*.html" --no-progress
 
          '''
 
        }
 
      }
 
    }
 
  }
 
}