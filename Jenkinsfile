pipeline {
    agent any

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
                    credentialsId: 'a55f9604-687f-4ad2-b248-66915d8f6a45'

                script {
                    echo "Rama seleccionada: ${params.BRANCH}"
                }
            }
        }

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

        stage('Get Config') {
            steps {
                script {
                    def configBranch = (params.BRANCH == 'develop') ? 'staging' : 'production'
                    sh """
                        curl -s -o samconfig.toml https://raw.githubusercontent.com/MsYapy/todo-list-aws-config/${configBranch}/samconfig.toml
                        echo "Configuración descargada desde rama: ${configBranch}"
                        cat samconfig.toml
                    """
                }
            }
        }

        stage('Deploy Staging') {
            when {
                expression { params.BRANCH == 'develop' }
            }
            steps {
                sh 'sam validate --region us-east-1'
                sh 'sam build'
                sh 'sam deploy --config-env staging --no-fail-on-empty-changeset'
            }
        }

        stage('Rest Test Staging') {
            when {
                expression { params.BRANCH == 'develop' }
            }
            steps {
                script {
                    env.BASE_URL = sh(
                        script: "aws cloudformation describe-stacks --stack-name resCP14Yapy-staging --query 'Stacks[0].Outputs[?OutputKey==`BaseUrlApi`].OutputValue' --region us-east-1 --output text",
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
                withCredentials([usernamePassword(credentialsId: 'a55f9604-687f-4ad2-b248-66915d8f6a45', usernameVariable: 'GIT_USER', passwordVariable: 'GIT_PASS')]) {
                    sh '''
                        git config user.email "jenkins@ci.local"
                        git config user.name "Jenkins CI"

                        rm -f bandit.out flake8.out

                        git fetch origin master
                        git checkout master || git checkout -b master origin/master

                        git show HEAD:Jenkinsfile > /tmp/Jenkinsfile.master.bak

                        git merge develop --no-edit --no-ff || {
                            git checkout --theirs . 2>/dev/null || true
                            git add -A
                            git commit --no-edit -m "Merge branch 'develop'"
                        }

                        cp /tmp/Jenkinsfile.master.bak Jenkinsfile
                        git add Jenkinsfile
                        git commit --amend --no-edit || true

                        git config credential.helper store
                        echo "https://${GIT_USER}:${GIT_PASS}@github.com" > ~/.git-credentials
                        git push origin master
                        rm -f ~/.git-credentials
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
