pipeline {
    agent any

    tools {
        nodejs "Node24"
    }

    environment {
        PROJECT_NAME = "longttworkshop2"
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
                echo 'Code đã được checkout thành công'
            }
        }

        stage('Build') {
            steps {
                echo 'Bắt đầu quá trình build...'
                sh 'npm install'
                echo 'Build hoàn thành!'
            }
        }

        stage('Lint & Test') {
            steps {
                echo 'Bắt đầu quá trình lint và test...'
                sh 'npm run test:ci'
                echo 'Lint và test hoàn thành!'
            }
        }

        stage('Deploy') {
            steps {
                echo "===== DEPLOY TO FIREBASE ====="
                sh 'npx firebase --version'

                withCredentials([string(credentialsId: 'LEGACY_TOKEN', variable: 'FIREBASE_TOKEN')]) {
                    sh '''
                        export FIREBASE_TOKEN="$FIREBASE_TOKEN"
                        npm run deploy:legacy -- --project=${PROJECT_NAME}
                    '''
                }
                echo 'Deploy hoàn thành!'
            }
        }
        stage('Deploy to Remote Host') {
            steps {
                echo '===== DEPLOY TO REMOTE HOST ====='
                withCredentials([sshUserPrivateKey(credentialsId: 'NEWBIE_SSH_KEY', keyFileVariable: 'SSH_KEY', usernameVariable: 'SSH_USER')]) {
                    script {
                        def releaseDir = env.RELEASE_DIR ?: new Date().format('yyyyMMdd')
                        def remoteBase = "/home/newbie/longtt/template2"
                        def localSrc = "${env.WORKSPACE}"
                        sh """
                            set -e
                            # Tạo thư mục deploy mới trên remote
                            ssh -i "$SSH_KEY" -o StrictHostKeyChecking=no $SSH_USER@10.1.1.195 "mkdir -p $remoteBase/deploy/$releaseDir && rm -rf $remoteBase/deploy/current"
                            # Copy toàn bộ source sang remote deploy
                            rsync -avz -e "ssh -i $SSH_KEY -o StrictHostKeyChecking=no" --exclude deploy --exclude .git ./web-performance-project1-initial/ $SSH_USER@10.1.1.195:$remoteBase/deploy/$releaseDir/
                            # Tạo symlink current
                            ssh -i "$SSH_KEY" -o StrictHostKeyChecking=no $SSH_USER@10.1.1.195 "ln -sfn $remoteBase/deploy/$releaseDir $remoteBase/deploy/current"
                        """
                    }
                }
                echo 'Deploy lên remote host hoàn thành!'
            }
        }
    }

    post {
        always {
            echo 'Pipeline đã hoàn thành!'
        }
        success {
            echo 'Pipeline chạy thành công!'
        }
        failure {
            echo 'Pipeline gặp lỗi!'
        }
    }
}
