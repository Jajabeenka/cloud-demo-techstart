// GitHub push -> webhook -> Jenkins -> Dynamic AWS fetch -> SSH/rsync -> Apache servers.
// Uses the SSH Agent plugin + 'webservers-ssh-key'. Assumes Jenkins has AWS CLI access.

pipeline {
    agent any

    options {
        timestamps()
        disableConcurrentBuilds()
    }

    environment {
        // ---- CONFIGURATION ----
        LB_NAME = 'milestone-lb' // The name parsed from your DNS
        DOCROOT = '/var/www/html'
        APP_SRC = './'
        // --------------------
    }

    stages {
        stage('Checkout') {
            steps { checkout scm }
        }

        stage('Build & Test') {
            steps {
                sh 'echo "No build step — deploying repo as-is."'
            }
        }

        stage('Deploy') {
            steps {
                sshagent(credentials: ['webservers-ssh-key']) {
                    sh '''
                        set -eu
                        SSH_OPTS="-o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null"
                        
                        echo "Fetching target instances behind the load balancer..."
                        
                        # 1. Get the Target Group ARN associated with the Load Balancer
                        TG_ARN=$(aws elbv2 describe-target-groups \
                            --load-balancer-arn $(aws elbv2 describe-load-balancers --names "${LB_NAME}" --query "LoadBalancers[0].LoadBalancerArn" --output text) \
                            --query "TargetGroups[0].TargetGroupArn" --output text)

                        # 2. Get the Instance IDs registered to that Target Group
                        INSTANCE_IDS=$(aws elbv2 describe-target-health --target-group-arn "$TG_ARN" --query "TargetHealthDescriptions[].Target.Id" --output text)

                        # 3. Resolve those Instance IDs to their Public (or Private) IP addresses
                        SERVERS=$(aws ec2 describe-instances --instance-ids $INSTANCE_IDS --query "Reservations[].Instances[].PublicIpAddress" --output text)

                        # 4. Loop through the dynamically discovered IPs and deploy
                        for IP in ${SERVERS}; do
                            HOST="ubuntu@${IP}"
                            echo "=== Deploying to ${HOST}:${DOCROOT} ==="
                            
                            rsync -az --delete -e "ssh ${SSH_OPTS}" --rsync-path="sudo rsync" \
                                --exclude '.git' --exclude 'Jenkinsfile' \
                                "${APP_SRC}" "${HOST}:${DOCROOT}/"
                                
                            ssh ${SSH_OPTS} "${HOST}" "sudo systemctl reload apache2"
                            echo "=== ${HOST} updated ==="
                        done
                    '''
                }
            }
        }
    }
}