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
                sh '''
                    set -e
                    for f in *.yaml; do
                        echo "Validating $f"
                        KUBECONFIG=/var/jenkins_home/kubeconfig /var/jenkins_home/bin/kubectl apply --dry-run=client -f "$f"
                    done
                '''
            }
        }
        stage('Deploy') {
            steps {
                sh '''
                    set -e
                    for f in *.yaml; do
                        echo "Deploying $f"
                        KUBECONFIG=/var/jenkins_home/kubeconfig /var/jenkins_home/bin/kubectl apply -f "$f"
                    done
                '''
            }
        }
    }
}
