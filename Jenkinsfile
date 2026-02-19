pipeline {
    agent any

    parameters {
        choice(name: 'BRANCH', choices: ['develop', 'master'], description: 'Rama a ejecutar')
    }

    stages {
        // 1) Obtener código
        stage('Get Code') {
            steps {
                git branch: "${params.BRANCH}", url: 'https://github.com/MsYapy/devOpsAws.git', credentialsId: 'yy'
                script {
                    echo "Rama seleccionada: ${params.BRANCH}"
                }
            }
        }

        // =============================================
        // STAGES CI - Solo rama develop
        // =============================================

        // 2) Pruebas estáticas (solo develop)
        stage('Static Analysis') {
            when { expression { params.BRANCH == 'develop' } }
            steps {
                sh '''bandit --exit-zero -r src/ -f custom -o bandit.out --msg-template "{abspath}:{line}: [{test_id}] {msg}"'''
                recordIssues tools: [pyLint(name: 'Bandit', pattern: 'bandit.out')]

                sh 'flake8 --exit-zero --format=pylint src/ > flake8.out'
                recordIssues tools: [flake8(name: 'flake8', pattern: 'flake8.out')]
            }
        }

        // 3) Deploy a staging (solo develop)
        stage('Deploy Staging') {
            when { expression { params.BRANCH == 'develop' } }
            steps {
                sh 'sam validate --region us-east-1'
                sh 'sam build'
                sh '''sam deploy \
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

        // 4) Tests de integración en staging (solo develop)
        stage('Rest Test Staging') {
            when { expression { params.BRANCH == 'develop' } }
            steps {
                script {
                    env.BASE_URL = sh(
                        script: "aws cloudformation describe-stacks --stack-name resCP14yapy-staging --query 'Stacks[0].Outputs[?OutputKey==`BaseUrlApi`].OutputValue' --region us-east-1 --output text",
                        returnStdout: true
                    ).trim()
                    echo "BASE_URL: ${env.BASE_URL}"
                }
                sh 'pytest test/integration/todoApiTest.py'
            }
        }

        // 5) Promote - merge a master (solo develop)
        stage('Promote') {
            when { expression { params.BRANCH == 'develop' } }
            steps {
                script {
                    sh "sed -i 's/\\[1.0.0\\] - 2021-01-08/[1.0.1] - 2021-01-08/g' CHANGELOG.md"
                    sh '''
                        git config user.email "jenkins@ci.local"
                        git config user.name "Jenkins CI"
                        git add CHANGELOG.md
                        git commit -m "Release 1.0.1"
                        git checkout master
                        git merge development --no-edit -X theirs
                        git checkout development -- Jenkinsfile
                        git commit --amend --no-edit
                        git push origin master
                    '''
                }
            }
        }

        // =============================================
        // STAGES CD - Solo rama master
        // =============================================

        // 6) Deploy a producción (solo master)
        stage('Deploy Production') {
            when { expression { params.BRANCH == 'master' } }
            steps {
                sh 'sam validate --region us-east-1'
                sh 'sam build'
                sh '''sam deploy \
                        --stack-name todo-list-aws-production \
                        --region us-east-1 \
                        --parameter-overrides Stage=production \
                        --capabilities CAPABILITY_IAM \
                        --no-disable-rollback \
                        --resolve-s3 \
                        --no-fail-on-empty-changeset
                    '''
            }
        }

        // 7) Tests de integración en producción (solo master)
        stage('Rest Test Production') {
            when { expression { params.BRANCH == 'master' } }
            steps {
                script {
                    env.BASE_URL = sh(
                        script: "aws cloudformation describe-stacks --stack-name todo-list-aws-production --query 'Stacks[0].Outputs[?OutputKey==`BaseUrlApi`].OutputValue' --region us-east-1 --output text",
                        returnStdout: true
                    ).trim()
                    echo "BASE_URL: ${env.BASE_URL}"
                }
                sh 'pytest test/integration/todoApiTest.py'
            }
        }
    }

    post {
        always {
            cleanWs()
        }
    }
}
