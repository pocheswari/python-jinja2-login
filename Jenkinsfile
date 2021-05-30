pipeline {
    parameters {
        string(name: 'git_user', defaultValue: 'kodekolli', description: 'Enter github username')
        choice(name: 'action', choices: 'deploy\ndelete', description: 'Action to deploy or delete sample app to EKS cluster')
    }

    agent any
    environment {
        VAULT_TOKEN = credentials('vault_token')
        USER_CREDENTIALS = credentials('DockerHub')
        registryCredential = 'DockerHub'
        dockerImage = ''
    }

    stages {
        stage('Retrieve SONAR credentials from vault'){
            when { expression { params.action == 'deploy' } }
            steps {
                script {
                    def host=sh(script: 'curl http://169.254.169.254/latest/meta-data/public-ipv4', returnStdout: true)
                    echo "$host"
                    sh "export VAULT_ADDR=http://${host}:8200"
                    sh 'export VAULT_SKIP_VERIFY=true'
                    sh "curl --header 'X-Vault-Token: ${VAULT_TOKEN}' --request GET http://${host}:8200/v1/MY_CREDS/data/secret > mycreds.json"
                    sh 'cat mycreds.json | jq -r .data.data.sonar_token > sonar_token.txt'
                    SONAR_TOKEN = readFile('sonar_token.txt').trim()            
                }
            }
        }
        stage('checkout') {
            steps {
                checkout scm       
            }
        }
        stage('Code Quality Check - SonarQube testing') {
            when { expression { params.action == 'deploy' } }
            when { branch 'development' } 
            steps {
                script {
                    dir('python-jinja2-login'){
                        def host=sh(script: 'curl http://169.254.169.254/latest/meta-data/public-ipv4', returnStdout: true)
                        echo "$host"
                        git url:"https://github.com/${params.git_user}/python-jinja2-login.git", branch:'development'
                        sh "/opt/sonarscanner/bin/sonar-scanner \
                        -Dsonar.projectKey=python-login \
                        -Dsonar.projectBaseDir=$WORKSPACE/python-jinja2-login \
                        -Dsonar.sources=. \
                        -Dsonar.language=py \
                        -Dsonar.host.url=http://${host}:9000 \
                        -Dsonar.login=${SONAR_TOKEN}"                        
                    }
                }
            }
            post {
                success {
                    echo "Code Quality check is completed"
                }
                failure {
                    echo "Code Quality check failed, vulnerabilities found"
                }
            }
        }
        stage('Source-Composition-Analysis'){
            when { expression { params.action == 'deploy' } }
            when { branch 'development' }
            steps {
                dir('python-jinja2-login') {
                    sh 'rm owasp* || true'
                    sh 'wget "https://raw.githubusercontent.com/kodekolli/python-jinja2-login/development/owasp-dependency-check.sh" '
                    sh 'chmod +x owasp-dependency-check.sh'
                    sh 'bash owasp-dependency-check.sh'
                    sh 'cat /var/lib/jenkins/OWASP-Dependency-Check/reports/dependency-check-report.xml'
                }
            }
        }
        stage('check Git secrets'){
            when { branch 'development' }
            steps {
                sh 'rm trufflehog || true'
                sh "docker run gesellix/trufflehog --json https://github.com/${params.git_user}/python-jinja2-login.git > trufflehog"
                sh 'cat trufflehog'
            }
        }
        stage('Build docker image and scan vulnerabilities'){
            when { expression { params.action == 'deploy' } }
            when { branch 'development' }
            steps {
                script {
                    sh 'printenv'
                    dir('python-jinja2-login'){
                        echo "Building docker image"                        
                        dockerImage = docker.build("${USER_CREDENTIALS_USR}/eks-demo-lab:${env.BUILD_ID}")
                        echo "Scanning the image for vulnerabilities"
                        echo dockerImage.id
                        sh "trivy image --severity HIGH,CRITICAL ${dockerImage.id}"
                    }
                }
            }
            post {
                success {
                    echo "Image scan check is completed"
                }
                failure {
                    echo "Image scan check failed, vulnerabilities found"
                }
            }
        }
        stage('Repo obfuscation using pyarmor'){
            when { expression { params.action == 'deploy' } }
            when { branch 'development' } 
            steps {
                script{
                    echo "Obfuscating the repo"
                    sh 'pyarmor obfuscate app.py'                    
                }
            }
            post {
                success {
                    echo "Sample app deployed to Dev EKS cluster."
                }
                failure {
                    echo "Sample app deployment failed to Dev EKS cluster."
                }
            }
        }
        stage('Deploying sample application to Dev EKS cluster') {
            when { expression { params.action == 'deploy' } }
            when { branch 'development' } 
            steps {
                script{
                    echo "Deploy app to EKS cluster"
                    sh 'ansible-playbook python-app.yml --user jenkins -e action=present -e config=$HOME/.kube/devconfig'
                    sleep 10
                    sh 'export APPELB=$(kubectl get svc -n default helloapp-svc -o jsonpath="{.status.loadBalancer.ingress[0].hostname}")'
                }
            }
            post {
                success {
                    echo "Sample app deployed to Dev EKS cluster."
                }
                failure {
                    echo "Sample app deployment failed to Dev EKS cluster."
                }
            }
        }
        stage('Delete sample application from Dev EKS cluster'){
            when { expression { params.action == 'delete' } }
            steps {
                echo "Deleting the app from EKS cluster"
                sh 'ansible-playbook python-app.yml --user jenkins -e action=absent -e config=$HOME/.kube/devconfig'
            }
        }
    }
}
