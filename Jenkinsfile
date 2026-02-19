pipeline {
    agent any

    stages {
        // 1) Obtener código
        stage('Get Code') {
            steps {
                // Multibranch: Jenkins detecta la rama automáticamente
                checkout scm
            }
        }

        // =============================================
        // STAGES CI - Solo rama develop
        // =============================================

        // 2) Pruebas estáticas (solo develop)
        stage('Static Analysis') {
            when { branch 'develop' }
            steps {
                sh '''bandit --exit-zero -r src/ -f custom -o bandit.out --msg-template "{abspath}:{line}: [{test_id}] {msg}"'''
                recordIssues tools: [pyLint(name: 'Bandit', pattern: 'bandit.out')]

                sh 'flake8 --exit-zero --format=pylint src/ > flake8.out'
                recordIssues tools: [flake8(name: 'flake8', pattern: 'flake8.out')]
            }
        }

        // 3) Deploy a staging (solo develop)
        stage('Deploy Staging') {
            when { branch 'develop' }
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
            when { branch 'develop' }
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
            when { branch 'develop' }
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
            when { branch 'master' }
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
            when { branch 'master' }
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

