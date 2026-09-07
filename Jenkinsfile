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
        stage('Validate') {
            steps {
                sh 'KUBECONFIG=/var/jenkins_home/kubeconfig /var/jenkins_home/bin/kubectl apply --dry-run=client -f snipeit-deployment.yaml'
            }
        }
        stage('Deploy') {
            steps {
                sh 'KUBECONFIG=/var/jenkins_home/kubeconfig /var/jenkins_home/bin/kubectl apply -f snipeit-deployment.yaml'
            }
        }
    }
}