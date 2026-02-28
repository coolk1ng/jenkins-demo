pipeline {
    agent any

    environment {
        // 项目相关变量
        PROJECT_NAME = 'jenkins-demo'      // 应用名称
        JAR_FILE = 'target/*.jar'               // Maven 打包生成的 jar 路径模式
        REMOTE_DIR = '/opt/apps/jenkins-demo' // 远程服务器部署目录
        REMOTE_SERVER = 'root@101.35.131.124'    // 远程服务器地址，格式 user@host
        // 如果使用 SSH 密钥，建议在 Jenkins 中配置凭据 ID，然后在 stage 中引用
    }

    stages {
        stage('Checkout') {
            steps {
                // 从代码仓库拉取代码（通常由 Jenkins 作业配置的 SCM 自动完成）
                checkout scm
            }
        }

        stage('Build') {
            steps {
                // 使用 Maven 打包，跳过测试（可选，根据需要调整）
                sh 'mvn clean package -DskipTests'
            }
            post {
                success {
                    // 构建成功后归档 JAR 文件，方便在 Jenkins 任务页面查看
                    archiveArtifacts artifacts: "${JAR_FILE}", fingerprint: true
                }
            }
        }

        stage('Upload JAR') {
            steps {
                script {
                    // 使用 sshagent 将本地 JAR 文件通过 SCP 传输到远程服务器
                    // 你需要事先在 Jenkins 中添加一个 SSH 私钥凭据，并将凭据 ID 替换到下面
                    sshagent(['ssh-jenkins-key']) {
                        sh """
                            # 确保远程目录存在
                            ssh ${REMOTE_SERVER} "mkdir -p ${REMOTE_DIR}"
                            # 上传 JAR 文件（覆盖已有文件）
                            scp ${JAR_FILE} ${REMOTE_SERVER}:${REMOTE_DIR}/${PROJECT_NAME}.jar
                        """
                    }
                }
            }
        }

        stage('Deploy') {
            steps {
                script {
                    sshagent(['ssh-jenkins-key']) {
                        sh """
                            ssh ${REMOTE_SERVER} "
                                # 进入部署目录
                                cd ${REMOTE_DIR}

                                # 停止旧进程（通过 kill 命令，可根据实际进程管理方式调整）
                                # 假设进程名包含项目名，用 pkill 停止
                                pkill -f ${PROJECT_NAME}.jar || true

                                # 等待进程完全退出（可选）
                                sleep 5

                                # 启动新进程（后台运行，并将日志输出到文件）
                                nohup java -jar ${PROJECT_NAME}.jar > app.log 2>&1 &

                                # 可选：检查进程是否启动成功
                                sleep 3
                                pgrep -f ${PROJECT_NAME}.jar && echo 'Application started successfully.' || echo 'Startup may have failed.'
                            "
                        """
                    }
                }
            }
        }
    }

    post {
        failure {
            // 构建失败时发送通知等
            echo 'Pipeline failed!'
        }
        always {
            // 清理工作空间等
            cleanWs()
        }
    }
}