pipeline {
    agent any

    environment {
        EC2_USER = 'ec2-user'
        EC2_HOST = '16.171.225.131'
        APP_DIR = '/var/www/html'
        REPO_URL = 'https://github.com/UshaAdiga/HTMLass6.git'
        BACKUP_DIR = '/home/ec2-user/backups'
    }

    stages {
        stage('Clone Repository') {
            steps {
                script {
                    def latestTag = sh(script: "git ls-remote --tags $REPO_URL | awk -F/ '{print $3}' | sort -V | tail -1", returnStdout: true).trim()
                    echo "Deploying version: $latestTag"
                    sh "git clone --branch $latestTag $REPO_URL app"
                }
            }
        }

        stage('Backup Current Version') {
            steps {
                sshagent(['EC2_SSH_KEY']) {
                    sh "ssh -o StrictHostKeyChecking=no $EC2_USER@$EC2_HOST 'bash /home/ec2-user/backup.sh'"
                }
            }
        }

        stage('Deploy New Version') {
            steps {
                sshagent(['EC2_SSH_KEY']) {
                    script {
                        try {
                            sh """
                                scp -o StrictHostKeyChecking=no -r app/* $EC2_USER@$EC2_HOST:$APP_DIR
                                ssh -o StrictHostKeyChecking=no $EC2_USER@$EC2_HOST 'sudo systemctl restart httpd'
                            """
                            echo "Deployment Successful!"
                        } catch (Exception e) {
                            error("Deployment Failed! Initiating Rollback...")
                        }
                    }
                }
            }
        }
    }

    post {
        failure {
            echo "Deployment failed! Rolling back..."
            sshagent(['EC2_SSH_KEY']) {
                sh """
                    ssh -o StrictHostKeyChecking=no $EC2_USER@$EC2_HOST '
                        LATEST_BACKUP=$(ls -t $BACKUP_DIR | head -1);
                        if [ ! -z "$LATEST_BACKUP" ]; then
                            tar -xzvf $BACKUP_DIR/$LATEST_BACKUP -C $APP_DIR;
                            sudo systemctl restart httpd;
                            echo "Rollback Successful!";
                        else
                            echo "No backup found! Cannot rollback.";
                        fi
                    '
                """
            }
        }
        success {
            echo "Deployment completed successfully!"
        }
    }
}
