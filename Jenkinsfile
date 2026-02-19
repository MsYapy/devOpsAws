pipeline {
    agent any

    parameters {
        choice(name: 'BRANCH', choices: ['develop', 'master'], description: 'Rama a ejecutar')
    }

    stages {

        // 1) Obtener código
        stage('Get Code') {
            steps {
                git branch: "${params.BRANCH}", 
                    url: 'https://github.com/MsYapy/devOpsAws.git', 
                    credentialsId: 'yy'
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
                    sh '''
                        # 1. Usar SSH en el repo
                        git remote set-url origin git@github.com:MsYapy/devOpsAws.git

                        # 2. Configurar identidad
                        git config user.email "jenkins@ci.local"
                        git config user.name "Jenkins CI"
                        git config merge.ours.driver true

                        # 3. Traer info fresca
                        git fetch origin master

                        # 4. Ir a master
                        git checkout master || git checkout -b master origin/master

                        # 5. Merge forzado con commit
                        git merge develop --no-edit --no-ff

                        # 6. Push
                        git push origin master
                    '''
                }
            }
        }

    }

    post {
        always {
            cleanWs()
        }
    }
}
