pipeline {
    agent any
    // Production

    stages {
        stage('Get Code') {
            steps {
                git branch: 'master',
                    url: 'https://github.com/MsYapy/devOpsAws.git',
                    credentialsId: 'yy'
            }
        }

        stage('Get Config') {
            steps {
                sh '''
                    curl -s -o samconfig.toml https://raw.githubusercontent.com/MsYapy/todo-list-aws-config/production/samconfig.toml
                    echo "Configuración descargada desde rama: production"
                    cat samconfig.toml
                '''
            }
        }

        stage('Deploy Production') {
            steps {
                sh 'sam validate --region us-east-1'
                sh 'sam build'
                sh 'sam deploy --config-env production --no-fail-on-empty-changeset'
            }
        }

        stage('Rest Test Production') {
            steps {
                script {
                    env.BASE_URL = sh(
                        script: "aws cloudformation describe-stacks --stack-name resCP14Yapy-production --query 'Stacks[0].Outputs[?OutputKey==`BaseUrlApi`].OutputValue' --region us-east-1 --output text",
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
