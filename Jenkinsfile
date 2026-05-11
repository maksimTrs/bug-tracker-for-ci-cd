pipeline {
    agent any

    stages {
        stage('Unit Tests - Backend') {
            agent {
                docker {
                    image 'snakee/golang-junit:1.21'
                    reuseNode true
                    args '-u root'
                }
            }
            steps {
                dir('bugtracker-backend') {
                    sh 'go test -v ./...'
                }
            }
        }
    }
}
