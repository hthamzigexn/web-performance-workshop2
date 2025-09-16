pipeline {
    agent any

    parameters {
        choice(name: 'DEPLOY_ENV', choices: ['local', 'remote', 'firebase', 'all'], description: 'Select deployment environment')
        string(name: 'USERNAME', defaultValue: 'htham', description: 'Username for deployment directory')
    }

    environment {
        FIREBASE_TOKEN = credentials('firebase-token')
        FIREBASE_PROJECT = 'jenkinsfirebaseht'
        LOCAL_SERVER = 'local-centos-server'
        REMOTE_SERVER = '118.69.34.46'
        REMOTE_PORT = '3334'
        DEPLOY_PATH = "/usr/share/nginx/html/jenkins/${params.USERNAME}2"
        REPO_NAME = 'web-performance-workshop2'
        BUILD_START_TIME = new Date().getTime()
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build') {
            steps {
                sh 'npm install'
            }
        }

        stage('Test') {
            steps {
                sh 'npm run test:ci'
            }
        }

        stage('Deploy to Firebase') {
            when {
                expression { params.DEPLOY_ENV == 'firebase' || params.DEPLOY_ENV == 'all' }
            }
            steps {
                sh 'export NODE_OPTIONS="--max-old-space-size=4096" && firebase deploy --token "$FIREBASE_TOKEN" --only hosting --project=$FIREBASE_PROJECT'
            }
        }

        stage('Deploy to Local Server') {
            when {
                expression { params.DEPLOY_ENV == 'local' || params.DEPLOY_ENV == 'all' }
            }
            steps {
                script {
                    deployToServer(env.LOCAL_SERVER, "", false)
                }
            }
        }

        stage('Deploy to Remote Server') {
            when {
                expression { params.DEPLOY_ENV == 'remote' || params.DEPLOY_ENV == 'all' }
            }
            steps {
                script {
                    deployToServer(env.REMOTE_SERVER, env.REMOTE_PORT, true)
                }
            }
        }
    }

    post {
        always {
            script {
                def BUILD_END_TIME = new Date().getTime()
                def durationMillis = BUILD_END_TIME - BUILD_START_TIME.toLong()
                env.BUILD_DURATION = (durationMillis / 1000).intValue()
                env.GIT_BRANCH = sh(script: 'git rev-parse --abbrev-ref HEAD || echo "unknown"', returnStdout: true).trim()
                env.GIT_COMMIT_SHORT = sh(script: 'git rev-parse --short HEAD || echo "unknown"', returnStdout: true).trim()
            }
        }
        success {
            slackSend(
                color: 'good',
                message: """:white_check_mark: *SUCCESS* - Deployment completed!
                • *User:* ${params.USERNAME}
                • *Job:* ${env.JOB_NAME} #${env.BUILD_NUMBER}
                • *Branch:* ${env.GIT_BRANCH} (${env.GIT_COMMIT_SHORT})
                • *Duration:* ${env.BUILD_DURATION} seconds
                • *Environment:* ${params.DEPLOY_ENV}"""
            )
        }
        failure {
            slackSend(
                color: 'danger',
                message: """:x: *FAILED* - Deployment failed!
                • *User:* ${params.USERNAME}
                • *Job:* ${env.JOB_NAME} #${env.BUILD_NUMBER}
                • *Branch:* ${env.GIT_BRANCH} (${env.GIT_COMMIT_SHORT})
                • *Duration:* ${env.BUILD_DURATION} seconds
                • *Environment:* ${params.DEPLOY_ENV}
                • *Error:* Check Jenkins console for details"""
            )
        }
    }
}

// Helper function to deploy to a server
def deployToServer(String server, String port, Boolean isRemote) {
    String portParam = isRemote ? "-p ${port}" : ""
    String scpPortParam = isRemote ? "-P ${port}" : ""

    withCredentials([sshUserPrivateKey(credentialsId: 'ssh-key-id', keyFileVariable: 'SSH_KEY')]) {
        sh """
            # Step 1: Check if repository directory exists
            ssh -i \$SSH_KEY -o StrictHostKeyChecking=no ${portParam} newbie@${server} "
                # Check if the repository directory exists
                if [ ! -d '${env.DEPLOY_PATH}/${env.REPO_NAME}' ]; then
                    echo 'Repository directory not found: ${env.DEPLOY_PATH}/${env.REPO_NAME}'
                    exit 1
                fi

                # Update the repository
                cd ${env.DEPLOY_PATH}/${env.REPO_NAME}
                git pull
            "

            # Step 2: Create release directory with timestamp and deploy files
            RELEASE_DATE=\$(date +%Y%m%d)
            ssh -i \$SSH_KEY -o StrictHostKeyChecking=no ${portParam} newbie@${server} "
                # Create release directory
                mkdir -p ${env.DEPLOY_PATH}/deploy/\${RELEASE_DATE}

                # Copy only necessary files from source to release directory
                cp ${env.DEPLOY_PATH}/${env.REPO_NAME}/index.html ${env.DEPLOY_PATH}/deploy/\${RELEASE_DATE}/
                cp ${env.DEPLOY_PATH}/${env.REPO_NAME}/404.html ${env.DEPLOY_PATH}/deploy/\${RELEASE_DATE}/
                cp -r ${env.DEPLOY_PATH}/${env.REPO_NAME}/css ${env.DEPLOY_PATH}/deploy/\${RELEASE_DATE}/
                cp -r ${env.DEPLOY_PATH}/${env.REPO_NAME}/js ${env.DEPLOY_PATH}/deploy/\${RELEASE_DATE}/
                cp -r ${env.DEPLOY_PATH}/${env.REPO_NAME}/images ${env.DEPLOY_PATH}/deploy/\${RELEASE_DATE}/

                # Create or update symlink to current directory
                cd ${env.DEPLOY_PATH}/deploy/
                ln -sf \${RELEASE_DATE} current

                # Keep only 5 most recent directories
                ls -1d [0-9]* 2>/dev/null | sort -r | tail -n +6 | xargs rm -rf 2>/dev/null || true
            "
        """
    }
}
