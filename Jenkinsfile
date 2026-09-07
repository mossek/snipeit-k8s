pipeline {
    agent any
    triggers {
        pollSCM('H/5 * * * *')
    }
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        stage('Deploy') {
            steps {
                sh 'KUBECONFIG=/var/jenkins_home/kubeconfig /var/jenkins_home/bin/kubectl apply -f snipeit-deployment.yaml'
            }
        }
    }
}
