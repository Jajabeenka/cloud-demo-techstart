// GitHub push -> webhook -> Jenkins -> SSH/rsync -> Apache servers (/var/www/html).
// Uses the SSH Agent plugin + the 'webservers-ssh-key' credential. Put at repo ROOT.

pipeline {
    agent any

    options {
        timestamps()
        disableConcurrentBuilds()
    }

    environment {
        // ---- EDIT THESE ----
        SERVERS = 'ubuntu@172.31.4.88 ubuntu@172.31.91.224'  // your two web servers
        DOCROOT = '/var/www/html'                             // Apache default doc root
        APP_SRC = './'                                        // repo root; 'dist/' if you build
        // --------------------
    }

    stages {

        stage('Checkout') {
            steps { checkout scm }
        }

        stage('Build & Test') {
            steps {
                // Put real build/test commands here if any, e.g. sh 'npm ci && npm run build'
                sh 'echo "No build step — deploying repo as-is."'
            }
        }

        stage('Deploy') {
            steps {
                sshagent(credentials: ['webservers-ssh-key']) {
                    sh '''
                        set -eu
                        SSH_OPTS="-o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null"
                        for HOST in ${SERVERS}; do
                            echo "=== Deploying to ${HOST}:${DOCROOT} ==="
                            # --rsync-path="sudo rsync" lets rsync write to /var/www/html on the server.
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
