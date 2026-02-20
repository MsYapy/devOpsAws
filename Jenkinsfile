pipeline {
    agent any
  // Production
    parameters {
        choice(
            name: 'BRANCH',
            choices: ['develop', 'master'],
            description: 'Rama a ejecutar'
        )
    }

    stages {
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

        stage('Static Analysis') {
            when {
                expression { params.BRANCH == 'develop' }
            }
            steps {
                sh '''bandit --exit-zero -r src/ -f custom -o bandit.out --msg-template "{abspath}:{line}: [{test_id}] {msg}"'''
                recordIssues tools: [pyLint(name: 'Bandit', pattern: 'bandit.out')]

                sh 'flake8 --exit-zero --format=pylint src/ > flake8.out'
                recordIssues tools: [flake8(name: 'flake8', pattern: 'flake8.out')]
            }
        }

        stage('Deploy Staging') {
            when {
                expression { params.BRANCH == 'develop' }
            }
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
            when {
                expression { params.BRANCH == 'develop' }
            }
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

        stage('Promote') {
            when {
                expression { params.BRANCH == 'develop' }
            }
            steps {
                sshagent(['yy']) {
                    script {
                        sh '''
                            git remote set-url origin git@github.com:MsYapy/devOpsAws.git

                            git config user.email "jenkins@ci.local"
                            git config user.name "Jenkins CI"

                            git fetch origin master
                            git checkout master || git checkout -b master origin/master

                            # Merge con estrategia que mantiene archivos de master en conflictos
                            git merge develop --no-edit --no-ff -X ours || {
                                # Si aún hay conflictos, resolverlos manteniendo versión de master
                                git checkout --ours .gitattributes Jenkinsfile 2>/dev/null || true
                                git add -A
                                git commit --no-edit -m "Merge branch 'develop' (auto-resolved)"
                            }
                            git push origin master
                        '''
                    }
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
