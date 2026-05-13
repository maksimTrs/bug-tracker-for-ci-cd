pipeline {
    agent { label 'linux && docker' }

    options {
        timeout(time: 10, unit: 'MINUTES')
        timestamps()
        ansiColor('xterm')
        disableConcurrentBuilds(abortPrevious: true)
        buildDiscarder(logRotator(numToKeepStr: '5', artifactNumToKeepStr: '5'))
    }

    stages {
        stage('Unit Tests') {
            parallel {
                stage('Backend') {
                    agent {
                        docker {
                            // TODO: pin to immutable digest — run: docker inspect --format='{{index .RepoDigests 0}}' snakee/golang-junit:1.21
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
                            archiveArtifacts artifacts: 'bugtracker-backend/test-results.xml, bugtracker-backend/reports/**',
                                             allowEmptyArchive: true,
                                             onlyIfSuccessful: false
                        }
                    }
                }

                stage('Frontend') {
                    agent {
                        docker {
                            // TODO: pin to immutable digest — run: docker inspect --format='{{index .RepoDigests 0}}' node:20-alpine
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
                                allowMissing:          true,
                                alwaysLinkToLastBuild: true,
                                keepAll:               true,
                                reportDir:             'bugtracker-frontend/reports/coverage',
                                reportFiles:           'index.html',
                                reportName:            'Frontend Coverage Report'
                            ]
                            archiveArtifacts artifacts: 'bugtracker-frontend/test-results.xml, bugtracker-frontend/reports/**',
                                             allowEmptyArchive: true,
                                             onlyIfSuccessful: false
                        }
                    }
                }
            }
        }

        stage('Launch Application') {
            agent {
                docker {
                    // TODO: pin to immutable digest — run: docker inspect --format='{{index .RepoDigests 0}}' docker:27.5.1
                    image 'docker:27.5.1'
                    reuseNode true
                    // mount host Docker socket (DooD) so compose controls host-level containers
                    args '-v /var/run/docker.sock:/var/run/docker.sock -u 0'
                }
            }
            steps {
                // --wait implies detached mode and blocks until all services are healthy/running
                // --wait-timeout prevents the stage from hanging indefinitely if a service never becomes healthy
                sh 'docker compose up --build --wait --wait-timeout 60'
            }
        }

        stage('API Tests') {
            agent {
                docker {
                    // TODO: pin to immutable digest — run: docker inspect --format='{{index .RepoDigests 0}}' node:20-alpine
                    image 'node:20-alpine'
                    reuseNode true
                    // --network=host lets the container reach compose services via localhost ports
                    // no browser install needed — all tests use Playwright request fixture (pure HTTP)
                    args '--network=host'
                }
            }
            steps {
                dir('tests-api') {
                    sh 'npm ci'
                    sh 'npx playwright test'
                }
            }
            post {
                always {
                    junit testResults: 'tests-api/test-results/results.xml',
                          allowEmptyResults: false
                    publishHTML target: [
                        allowMissing:          true,   // playwright-report not written when Playwright fails to start — prevents double failure in post
                        alwaysLinkToLastBuild: true,
                        keepAll:               true,
                        reportDir:             'tests-api/playwright-report',
                        reportFiles:           'index.html',
                        reportName:            'API Tests Report'
                    ]
                    archiveArtifacts artifacts: 'tests-api/test-results/**, tests-api/playwright-report/**',
                                     allowEmptyArchive: true,
                                     onlyIfSuccessful: false
                }
            }
        }
    }

    post {
        always {
            // run cleanup inside docker:27.5.1 — Jenkins node has no Compose V2 plugin
            // --rmi local removes compose-built images; prune cleans dangling layers from previous builds
            // --remove-orphans removes containers left over from a previous run whose services no longer exist in compose file
            // TODO: pin docker:27.5.1 to immutable digest here as well (same image as Launch Application stage)
            script {
                docker.image('docker:27.5.1')
                      .inside('-v /var/run/docker.sock:/var/run/docker.sock -u 0') {
                    sh '''
                        exit_code=0
                        docker compose down --volumes --remove-orphans --rmi local || exit_code=$?
                        if [ "$exit_code" -ne 0 ]; then
                            echo "WARNING: docker compose down failed with exit code $exit_code"
                        fi
                    '''
                    sh '''
                        exit_code=0
                        docker image prune -f || exit_code=$?
                        if [ "$exit_code" -ne 0 ]; then
                            echo "WARNING: docker image prune failed with exit code $exit_code"
                        fi
                    '''
                }
            }
        }
        cleanup {
            cleanWs()
        }
    }
}
