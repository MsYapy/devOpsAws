pipeline {
    agent any
    // Production

    stages {
        stage('Deploy') {
            steps {
                git branch: 'master',
                    url: 'https://github.com/MsYapy/devOpsAws.git',
                    credentialsId: 'yy'
            }
        }

        stage('Deploy Production') {
            steps {
                sh 'sam validate --region us-east-1'
                sh 'sam build'
                sh '''
                    sam deploy \
                        --stack-name resCP14yapy-production \
                        --region us-east-1 \
                        --parameter-overrides Stage=production \
                        --capabilities CAPABILITY_IAM \
                        --no-disable-rollback \
                        --resolve-s3 \
                        --no-fail-on-empty-changeset
                '''
            }
        }

        stage('Rest Test Production') {
            steps {
                script {
                    env.BASE_URL = sh(
                        script: "aws cloudformation describe-stacks --stack-name resCP14yapy-production --query 'Stacks[0].Outputs[?OutputKey==`BaseUrlApi`].OutputValue' --region us-east-1 --output text",
                        returnStdout: true
                    ).trim()

                    echo "BASE_URL: ${env.BASE_URL}"
                }

                  // Production: SOLO tests de lectura (no modifica datos)
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
