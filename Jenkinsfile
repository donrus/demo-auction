pipeline {
    agent any
    options {
        timestamps()
    }
    environment {
        CI = 'true'
        REGISTRY = credentials("REGISTRY")
        IMAGE_TAG = sh(returnStdout: true, script: "echo '${env.BUILD_TAG}' | sed 's/%2F/-/g'").trim()
    }
    stages {
        stage("Init") {
            steps {
                sh "docker-compose down -v --remove-orphans"
                sh "docker run --rm -v ${PWD}/api:/app -w /app alpine sh -c 'rm -rf var/cache/* var/log/* var/test/*'"
                sh "docker run --rm -v ${PWD}/frontend:/app -w /app alpine sh -c 'rm -rf .ready build'"
                sh "docker run --rm -v ${PWD}/cucumber:/app -w /app alpine sh -c 'rm -rf var/*'"
                sh "docker-compose pull --include-deps"
                sh "docker-compose build"
                sh "docker-compose up -d"
                sh "docker run --rm -v ${PWD}/api:/app -w /app alpine chmod 777 var/cache var/log var/test"
                sh "docker-compose run --rm api-php-cli composer install"
                sh "docker-compose run --rm api-php-cli wait-for-it api-postgres:5432 -t 30"
                sh "docker-compose run --rm api-php-cli composer app migrations:migrate"
                sh "docker-compose run --rm api-php-cli composer app fixtures:load"
                sh "docker-compose run --rm frontend-node-cli yarn install"
                sh "docker run --rm -v ${PWD}/frontend:/app -w /app alpine touch .ready"
                sh "docker-compose run --rm cucumber-node-cli yarn install"
            }
        }
        stage("Valid") {
            steps {
                sh "docker-compose run --rm api-php-cli composer app orm:validate-schema"
            }
        }
        stage("Lint") {
            parallel {
                stage("API") {
                    steps {
                        sh "docker-compose run --rm api-php-cli composer lint"
                        sh "docker-compose run --rm api-php-cli composer cs-check"
                    }
                }
                stage("Frontend") {
                    steps {
                        sh "docker-compose run --rm frontend-node-cli yarn eslint"
                        sh "docker-compose run --rm frontend-node-cli yarn stylelint"
                    }
                }
                stage("Cucumber") {
                    steps {
                        sh "docker-compose run --rm cucumber-node-cli yarn lint"
                    }
                }
            }
        }
        stage("Analyze") {
            steps {
                sh "docker-compose run --rm api-php-cli composer psalm"
            }
        }
        stage("Test") {
            parallel {
                stage("API") {
                    steps {
                        sh "make api-test"
                    }
                    post {
                        failure {
                            archiveArtifacts 'api/var/log/**/*'
                        }
                    }
                }
                stage("Front") {
                    steps {
                        sh "make frontend-test"
                    }
                }
            }
        }
        stage("Down") {
            steps {
                sh "make docker-down-clear"
            }
        }
        stage("Build") {
            steps {
                sh "make build"
            }
        }
        stage("Testing") {
            stages {
                stage("Build") {
                    steps {
                        sh "make testing-build"
                    }
                }
                stage("Init") {
                    steps {
                        sh "make testing-init"
                    }
                }
                stage("Smoke") {
                    steps {
                        sh "make testing-smoke"
                    }
                    post {
                        failure {
                            archiveArtifacts 'cucumber/var/*'
                        }
                    }
                }
                stage("E2E") {
                    steps {
                        sh "make testing-e2e"
                    }
                    post {
                        failure {
                            archiveArtifacts 'cucumber/var/*'
                        }
                    }
                }
                stage("Down") {
                    steps {
                        sh "make testing-down-clear"
                    }
                }
            }
        }
        stage("Push") {
            when {
                branch "master"
            }
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'REGISTRY_AUTH',
                        usernameVariable: 'USER',
                        passwordVariable: 'PASSWORD'
                    )
                ]) {
                    sh "docker login -u=$USER -p='$PASSWORD' $REGISTRY"
                }
                sh "make push"
            }
        }
        stage ('Prod') {
            when {
                branch "master"
            }
            steps {
                withCredentials([
                    string(credentialsId: 'PRODUCTION_HOST', variable: 'HOST'),
                    string(credentialsId: 'PRODUCTION_PORT', variable: 'PORT'),
                    string(credentialsId: 'API_DB_PASSWORD', variable: 'API_DB_PASSWORD'),
                    string(credentialsId: 'API_MAILER_HOST', variable: 'API_MAILER_HOST'),
                    string(credentialsId: 'API_MAILER_PORT', variable: 'API_MAILER_PORT'),
                    string(credentialsId: 'API_MAILER_USER', variable: 'API_MAILER_USER'),
                    string(credentialsId: 'API_MAILER_PASSWORD', variable: 'API_MAILER_PASSWORD'),
                    string(credentialsId: 'API_MAILER_FROM_EMAIL', variable: 'API_MAILER_FROM_EMAIL'),
                    string(credentialsId: 'SENTRY_DSN', variable: 'SENTRY_DSN')
                ]) {
                    sshagent (credentials: ['PRODUCTION_AUTH']) {
                        sh "BUILD_NUMBER=${env.BUILD_NUMBER} make deploy"
                    }
                }
            }
        }
    }
    post {
        always {
            sh "make docker-down-clear || true"
            sh "make testing-down-clear || true"
            sh "make deploy-clean || true"
        }
        failure {
            emailext (
                subject: "FAIL Job ${env.JOB_NAME} ${env.BUILD_NUMBER}",
                body: "Check console output at: ${env.BUILD_URL}/console",
                recipientProviders: [[$class: 'DevelopersRecipientProvider']]
            )
        }
    }
}
