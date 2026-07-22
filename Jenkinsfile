pipeline {

    agent any

    environment {
        // Azure Storage
        STORAGE_ACCOUNT = "carranzastg"
        FILE_SHARE = "webcontent"

        // Staging (ACI)
        ACI_URL = "http://carranzalabel.centralindia.azurecontainer.io"

        // Production EC2 Private IPs
        WEB1 = "10.0.111.165"
        WEB2 = "10.0.0.80"

        REMOTE_DIR = "/var/www/html"
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Deploy to Azure Staging') {
            steps {
                withCredentials([
                    string(credentialsId: 'azure-storage-key', variable: 'STORAGE_KEY')
                ]) {
                    sh '''
                        echo "Uploading files to Azure File Share..."

                        az storage file upload-batch \
                            --account-name $STORAGE_ACCOUNT \
                            --account-key $STORAGE_KEY \
                            --destination $FILE_SHARE \
                            --source .
                    '''
                }
            }
        }

        stage('Wait for ACI') {
            steps {
                echo "Waiting for Azure File Share changes to be reflected..."
                sleep(time: 20, unit: 'SECONDS')
            }
        }

        stage('HTTP Availability Test') {
            steps {
                sh '''
                    STATUS=$(curl -o /dev/null -s -w "%{http_code}" $ACI_URL)

                    if [ "$STATUS" != "200" ]; then
                        echo "ERROR: ACI returned HTTP $STATUS"
                        exit 1
                    fi

                    echo "ACI is reachable."
                '''
            }
        }

        stage('Content Assertion') {
            steps {
                sh '''
                    PAGE=$(curl -s $ACI_URL)

                    echo "$PAGE"

                    # Change "Welcome" to text that exists on your homepage
                    echo "$PAGE" | grep -q "Welcome"

                    echo "Content validation passed."
                '''
            }
        }

        stage('Deploy to Production') {
            steps {
                sshagent(['webserver-id']) {
                    sh '''
                        echo "Deploying to Web Server 1..."

                        rsync -avz --delete \
                            --rsync-path="sudo rsync" \
                            --exclude='.git' \
                            --exclude='Jenkinsfile' \
                            -e "ssh -o StrictHostKeyChecking=no" \
                            ./ ubuntu@$WEB1:$REMOTE_DIR/

                        ssh -o StrictHostKeyChecking=no ubuntu@$WEB1 "
                            sudo chown -R www-data:www-data $REMOTE_DIR &&
                            sudo systemctl restart apache2
                        "

                        echo "Deploying to Web Server 2..."

                        rsync -avz --delete \
                            --rsync-path="sudo rsync" \
                            --exclude='.git' \
                            --exclude='Jenkinsfile' \
                            -e "ssh -o StrictHostKeyChecking=no" \
                            ./ ubuntu@$WEB2:$REMOTE_DIR/

                        ssh -o StrictHostKeyChecking=no ubuntu@$WEB2 "
                            sudo chown -R www-data:www-data $REMOTE_DIR &&
                            sudo systemctl restart apache2
                        "
                    '''
                }
            }
        }
    }

    post {

        success {
            echo "Pipeline completed successfully."
        }

        failure {
            echo "Pipeline failed. Production deployment was skipped."
        }

        always {
            cleanWs()
        }
    }
}
