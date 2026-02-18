pipeline {
    agent any
    
    stages {
        // 1) Traer el código de la rama development
        stage('Get Code') {
            steps {
                git branch: 'develop', url: 'https://github.com/MsYapy/devOpsAws'
            }
        }
        
        // 2) Pruebas estáticas
        stage('Static Analysis') {
            steps {
                sh '''bandit --exit-zero -r src/ -f custom -o bandit.out --msg-template "{abspath}:{line}: [{test_id}] {msg}"'''
                recordIssues tools: [pyLint(name: 'Bandit', pattern: 'bandit.out')]
                
                sh 'flake8 --exit-zero --format=pylint src/ > flake8.out'
                recordIssues tools: [flake8(name: 'flake8', pattern: 'flake8.out')]
            }
        }
        
        // 3) Despliegue SAM
        stage('Deploy') {
            steps {
                       sh '''
                    echo "=== Desplegando a Staging con SAM ==="

                    # Build de la aplicación SAM
                    sam validate --region us-east-1
                    sam build --template template.yaml

                    # Deploy a Staging
                    sam deploy \
                        --stack-name resCP14Yapy-staging \
                        --region us-east-1 \
                        --parameter-overrides Stage=staging \
                        --capabilities CAPABILITY_IAM \
                        --no-disable-rollback \
                        --resolve-s3 \
                        --no-fail-on-empty-changeset
                    '''
            }
        }
        
        // 4) Pruebas de integración
        stage('Rest Test') {
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
        
        // 5) Promote - Marcar versión como Release
        stage('Promote') {
            steps {
                script {
                    // Actualizar CHANGELOG.md
                    sh "sed -i 's/\\[1.0.0\\] - 2021-01-08/[1.0.1] - 2021-01-08/g' CHANGELOG.md"
                    
                    // Commit y merge a master sin incluir el Jenkinsfile
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
    }
    
    post {
        always {
            cleanWs()
        }
    }
}
