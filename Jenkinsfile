pipeline {
    parameters {
        string(name: 'git_user', defaultValue: 'kodekolli', description: 'Enter github username:')
    }

    agent any
    environment {
        USER_CREDENTIALS = credentials('DockerHub')
        registryCredential = 'DockerHub'
        dockerImage = ''
    }
    stages {
        stage('clone repo') {
            steps {
                git url:"https://github.com/${params.git_user}/eks-demo-project.git", branch:'test'
            }
        }
        stage('Deploying sample app to Test EKS cluster') {
            when {
                anyof {
                    branch 'test';
                    branch 'qa';
                    branch 'perf';
                    branch 'staging'
                }
            }       
            steps {
                script{
                    dir('python-jinja2-login'){
                        echo "Building docker image"
                        dockerImage = docker.build("${USER_CREDENTIALS_USR}/eks-demo-lab:${env.BUILD_ID}")
                        echo "Pushing the image to registry"
                        docker.withRegistry( 'https://registry.hub.docker.com', registryCredential ) {
                            dockerImage.push("latest")
                            dockerImage.push("${env.BUILD_ID}")
                        }
                        echo "Deploy app to EKS cluster"
                        sh 'ansible-playbook python-app.yml --user jenkins -e action=present -e config=$HOME/.kube/testconfig'
                        sleep 10
                        sh 'export APPELB=$(kubectl get svc -n default helloapp-svc -o jsonpath="{.status.loadBalancer.ingress[0].hostname}")'
                    }
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
        stage('DAST testing using OWASP ZAP') {
            steps {
                sh "mkdir -p ${pwd}/reports"
                sh "docker run --detach --name zap -v ${pwd}/reports:/zap/reports/:rw -i owasp/zap2docker-stable zap-baseline.py -t http://${APPELB}"
            }
        }
    }
}
