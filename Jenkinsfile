pipeline {
    agent none

    stages {
        stage('Unit Tests - Backend') {
            agent {
                docker {
                    image 'golang:1.21-alpine'
                    args '-v go-module-cache:/go/pkg/mod'
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
