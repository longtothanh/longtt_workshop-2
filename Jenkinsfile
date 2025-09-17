pipeline {
    agent any

    tools {
        nodejs "Node24"
    }

    environment {
        PROJECT_NAME = "longttworkshop2"
    }

    triggers {
        githubPush()
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
                    def releaseDir = new Date().format('yyyyMMddHHmmss')
                    def remoteBase = "/usr/share/nginx/html/jenkins/longtt"
                    def host = "10.1.1.195"

                    sh """
                    set -e
                    echo 'PWD:' && pwd
                    echo 'List workspace:' && ls -la

                    # Tạo gói deploy từ WORKSPACE, loại bỏ .git và node_modules nếu có
                    TAR=/tmp/site-${releaseDir}.tar.gz
                    tar --exclude='.git' --exclude='node_modules' -czf "\$TAR" -C "${env.WORKSPACE}" .

                    # Tạo thư mục release trên remote
                    ssh -i "${SSH_KEY}" -o StrictHostKeyChecking=no ${SSH_USER}@${host} \\
                        "mkdir -p ${remoteBase}/deploy/${releaseDir} && rm -rf ${remoteBase}/deploy/current"

                    # Upload gói
                    scp -i "${SSH_KEY}" -o StrictHostKeyChecking=no "\$TAR" \\
                        ${SSH_USER}@${host}:/tmp/site-${releaseDir}.tar.gz

                    # Giải nén & cập nhật symlink current
                    ssh -i "${SSH_KEY}" -o StrictHostKeyChecking=no ${SSH_USER}@${host} <<'EOF'
                        set -e
                        remoteBase="${remoteBase}"
                        releaseDir="${releaseDir}"
                        mkdir -p "\$remoteBase/deploy/\$releaseDir"
                        tar -xzf "/tmp/site-\${releaseDir}.tar.gz" -C "\$remoteBase/deploy/\$releaseDir"
                        ln -sfn "\$remoteBase/deploy/\$releaseDir" "\$remoteBase/deploy/current"
                        rm -f "/tmp/site-\${releaseDir}.tar.gz"
                    EOF
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
