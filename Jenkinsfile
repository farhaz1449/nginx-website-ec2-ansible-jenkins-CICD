pipeline {
    agent any

    stages {
        stage('Clone Code') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/farhaz1449/nginx-website-ec2-ansible-jenkins-CICD.git'
            }
        }
        stage('Run Ansible Playbook') {
            steps {
                ansiblePlaybook playbook: 'web-playbook.yml', inventory: 'hosts'
            }
        }
    }
    post {
        success {
            echo 'Pipeline execution successful.'
        }
        failure {
            echo 'Pipeline execution failed.'
        }
    }
}