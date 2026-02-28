pipeline {
    agent any

    // Staging (rama develop)
    stages {
        stage('Deploy') {
            steps {
                git branch: 'develop',
                    url: 'https://github.com/MsYapy/devOpsAws.git',
                    credentialsId: 'yy'
            }
        }

        // =============================================
        // STAGE CD - Staging
        // =============================================

        stage('Deploy Staging') {
            steps {
                sh 'sam validate --region us-east-1'
                sh 'sam build'
                sh '''
                    sam deploy \
                        --stack-name resCP14yapy-staging \
                        --region us-east-1 \
                        --parameter-overrides Stage=staging \
                        --capabilities CAPABILITY_IAM \
                        --no-disable-rollback \
                        --resolve-s3 \
                        --no-fail-on-empty-changeset
                '''
            }
        }

        stage('Rest Test Staging') {
            steps {
                script {
                    env.BASE_URL = sh(
                        script: "aws cloudformation describe-stacks --stack-name resCP14yapy-staging --query 'Stacks[0].Outputs[?OutputKey==`BaseUrlApi`].OutputValue' --region us-east-1 --output text",
                        returnStdout: true
                    ).trim()

                    echo "BASE_URL Staging: ${env.BASE_URL}"
                }

                sh 'pytest test/integration/todoApiTest.py -m readonly'
            }
        }
    }

    post {
        always {
            cleanWs()
        }
    }
}
