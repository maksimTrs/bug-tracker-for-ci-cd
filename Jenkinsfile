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
                            sh '''
                                # -set-exit-code: without this go-junit-report exits 0 even when tests fail
                                # build would stay green
                                go test -v -coverprofile=coverage.out -covermode=atomic ./... 2>&1 \
                                    | go-junit-report -set-exit-code > test-results.xml

                                go tool cover -html=coverage.out -o coverage.html
                                mkdir -p reports
                                mv coverage.html reports/
                            '''
                        }
                    }
                    post {
                        always {
                            junit testResults: 'bugtracker-backend/test-results.xml',
                                  allowEmptyResults: false
                            publishHTML target: [
                                allowMissing:          true,   // coverage.html not written when tests fail — prevents double failure in post
                                alwaysLinkToLastBuild: true,
                                keepAll:               true,
                                reportDir:             'bugtracker-backend/reports',
                                reportFiles:           'coverage.html',
                                reportName:            'Backend Coverage Report'
                            ]
                        }
                    }
                }

                stage('Frontend') {
                    agent {
                        docker {
                            image 'node:20-alpine'
                            reuseNode true  // share workspace with outer agent — no separate checkout inside container
                        }
                    }
                    steps {
                        dir('bugtracker-frontend') {
                            sh '''
                                npm ci
                                npm test
                                rm -rf reports
                                mkdir -p reports
                                mv coverage reports/
                            '''
                        }
                    }
                    post {
                        always {
                            junit testResults: 'bugtracker-frontend/test-results.xml',
                                  allowEmptyResults: false
                            publishHTML target: [
                                allowMissing:          true,   // coverage dir not written when tests fail — prevents double failure in post
                                alwaysLinkToLastBuild: true,
                                keepAll:               true,
                                reportDir:             'bugtracker-frontend/reports/coverage',
                                reportFiles:           'index.html',
                                reportName:            'Frontend Coverage Report'
                            ]
                        }
                    }
                }
            }
        }
    }
}
