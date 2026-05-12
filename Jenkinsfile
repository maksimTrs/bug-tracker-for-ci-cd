pipeline {
    agent any

    stages {
        stage('Unit Tests') {
            parallel {
                stage('Backend') {
                    agent {
                        docker {
                            image 'snakee/golang-junit:1.21'
                            reuseNode true
                            args '-u root'
                        }
                    }
                    steps {
                        dir('bugtracker-backend') {
                            sh 'go test -v ./... 2>&1 | go-junit-report -set-exit-code > test-results.xml'
                        }
                    }
                    post {
                        always {
                            junit testResults: 'bugtracker-backend/test-results.xml',
                                  allowEmptyResults: false
                        }
                    }
                }

                stage('Frontend') {
                    agent {
                        docker {
                            image 'node:20-alpine'
                            reuseNode true
                        }
                    }
                    steps {
                        dir('bugtracker-frontend') {
                            sh '''
                                npm ci
                                npm test
                            '''
                        }
                    }
                    post {
                        always {
                            junit testResults: 'bugtracker-frontend/test-results.xml',
                                  allowEmptyResults: false
                        }
                    }
                }
            }
        }
    }
}
