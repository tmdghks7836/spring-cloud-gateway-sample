def mainDir="."
def ecrLoginHelper="docker-credential-ecr-login"
def region="ap-northeast-2"
def ecrUrl="238968007735.dkr.ecr.ap-northeast-2.amazonaws.com"
def serviceName="voucher-gateway"
def repository="${serviceName}-${PROJECT_SECTION}"
def ecsCluster="voucher-cluster-${PROJECT_SECTION}"
def ecsService="${serviceName}-${PROJECT_SECTION}"

pipeline {
    agent any

    tools {
        jdk('java17')
    }

    stages {
        stage('Pull Codes from Github'){
            steps{
                checkout scm
                script {
                    SLACK_CHANNEL = "#젠킨스_pipeline"
                    SLACK_SUCCESS_COLOR = "#2C953C";
                    SLACK_FAIL_COLOR = "#FF3232";
                    // Git Commit 계정
                    GIT_COMMIT_AUTHOR = sh(script: "git --no-pager show -s --format=%an ${env.GIT_COMMIT}", returnStdout: true).trim();
                    // Git Commit 메시지
                    GIT_COMMIT_MESSAGE = sh(script: "git --no-pager show -s --format=%B ${env.GIT_COMMIT}", returnStdout: true).trim();
                }
            }
            post {
                success {
                    slackSend (
                        channel: SLACK_CHANNEL,
                        color: SLACK_SUCCESS_COLOR,
                        message: "[${ecsService}] Pipeline Start!!!\n${env.JOB_NAME}(${env.BUILD_NUMBER})\n${GIT_COMMIT_AUTHOR} - ${GIT_COMMIT_MESSAGE}\n${env.BUILD_URL}"
                    )
                }
            }
        }
        stage('Build Codes by Gradle') {
            steps {
                sh """
                cd ${mainDir}
                ./gradlew clean
                ./gradlew bootJar
                """
            }
            post {
                success {
                    slackSend (
                        channel: SLACK_CHANNEL,
                        color: SLACK_SUCCESS_COLOR,
                        message: "[${ecsService}] Build Codes by Gradle Success"
                    )
                }
                failure {
                    slackSend (
                        channel: SLACK_CHANNEL,
                        color: SLACK_FAIL_COLOR,
                        message: "[${ecsService}] Build Codes by Gradle Failed"
                    )
                }
            }
        }

         stage('Build Docker Image by Jib & Push to AWS ECR Repository') {
             steps {
                 withAWS(region:"${region}", credentials:"aws-key") {
                     ecrLogin()
                         sh """
                         curl -O https://amazon-ecr-credential-helper-releases.s3.us-east-2.amazonaws.com/0.4.0/linux-amd64/${ecrLoginHelper}
                         chmod +x ${ecrLoginHelper}
                         mv ${ecrLoginHelper} /usr/local/bin/
                         cd ${mainDir}
                         ./gradlew jib -Djib.to.image=${ecrUrl}/${repository}:latest -Djib.console='plain'
                         """
                 }
             }
             post {
                 success {
                     slackSend (
                         channel: SLACK_CHANNEL,
                         color: SLACK_SUCCESS_COLOR,
                         message: "[${ecsService}] Push to AWS ECR Repository Success"
                     )
                 }
                 failure {
                     slackSend (
                         channel: SLACK_CHANNEL,
                         color: SLACK_FAIL_COLOR,
                         message: "[${ecsService}] Push to AWS ECR Repository Failed"
                     )
                 }
             }
         }

        stage('Deploy to AWS ECS VM'){
             steps {
                script{
                    try {
                        withAWS(region:'${region}', credentials:"aws-key") {
                            sh"""
                                aws ecs update-service --region ${region} --cluster ${ecsCluster} --service ${ecsService} --force-new-deployment
                            """
                        }
                    } catch (error) {
                        print(error)
                        echo 'Remove Deploy Files'
                        sh "rm -rf /var/lib/jenkins/workspace/${env.JOB_NAME}/*"
                        currentBuild.result = 'FAILURE'
                    }
                }
            }
            post {
                success {
                    slackSend (
                        channel: SLACK_CHANNEL,
                        color: SLACK_SUCCESS_COLOR,
                        message: "[${ecsService}] Deploy to AWS ECS VM Success"
                    )
                }
                failure {
                    slackSend (
                        channel: SLACK_CHANNEL,
                        color: SLACK_FAIL_COLOR,
                        message: "[${ecsService}] Deploy to AWS ECS VM Failed"
                    )
                }
            }
        }
    }
}
