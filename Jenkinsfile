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
            script {
                slackSend(
                    channel: env.SLACK_CHANNEL,
                    color: '#CCCCCC',
                    message: ":information_source: *Finished* — ${env.JOB_NAME} #${env.BUILD_NUMBER} (status: ${currentBuild.currentResult})\n${env.BUILD_URL}"
                )
            }
            }
        success {
            echo 'Pipeline chạy thành công!'
                script {
                    def author = sh(script: "git --no-pager log -1 --pretty=format:'%an'", returnStdout: true).trim()
                    def commitMsg = sh(script: "git --no-pager log -1 --pretty=%s", returnStdout: true).trim()
                    def commitTime = sh(script: "git --no-pager log -1 --date=iso --pretty=format:'%cd'", returnStdout: true).trim()
                    slackSend(
                        channel: env.SLACK_CHANNEL,
                        color: '#2EB67D',
                        message: """
     Deployment Successful! :tada:
     Author: ${author}
     Commit: ${commitMsg}
     Time: ${commitTime} UTC
     Links:
     • Firebase: https://diepttn-workshop2.web.app
     • Remote: http://118.69.34.46/jenkins/dieptrinh2/deploy/current/
     """
                    )
                }
        }
        failure {
            echo 'Pipeline gặp lỗi!'
            script {
                def tail = sh(script: "tail -n 50 ${env.WORKSPACE}/@tmp/durable-*/stdout 2>/dev/null || true", returnStdout: true).trim()
                slackSend(
                    channel: env.SLACK_CHANNEL,
                    color: '#E01E5A',
                    message: """
    :x: *FAILED* — ${env.JOB_NAME} #${env.BUILD_NUMBER}
    *Branch:* `${env.BRANCH_NAME ?: 'main'}`
    ${env.BUILD_URL}

    *Last 50 lines of log:*
    ```
    ${tail.take(1900)}
    ```
    """
                )
            }
        }
    }
}
