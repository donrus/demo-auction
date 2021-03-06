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
                sh "docker run --rm -v ${env.WORKSPACE}/api:/app -w /app alpine sh -c 'rm -rf var/cache/* var/log/* var/test/*'"
                sh "docker run --rm -v ${env.WORKSPACE}/frontend:/app -w /app alpine sh -c 'rm -rf .ready build'"
                sh "docker run --rm -v ${env.WORKSPACE}/cucumber:/app -w /app alpine sh -c 'rm -rf var/*'"
                sh "docker-compose pull --include-deps"
                sh "docker-compose build"
                sh "docker-compose up -d"
                sh "docker run --rm -v ${env.WORKSPACE}/api:/app -w /app alpine chmod 777 var/cache var/log var/test"
                sh "docker-compose run --rm api-php-cli composer install"
                sh "docker-compose run --rm api-php-cli wait-for-it api-postgres:5432 -t 30"
                sh "docker-compose run --rm api-php-cli composer app migrations:migrate"
                sh "docker-compose run --rm api-php-cli composer app fixtures:load"
                sh "docker-compose run --rm frontend-node-cli yarn install"
                sh "docker run --rm -v ${env.WORKSPACE}/frontend:/app -w /app alpine touch .ready"
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
                        sh "docker-compose run --rm api-php-cli composer test"
                    }
                    post {
                        failure {
                            archiveArtifacts 'api/var/log/**/*'
                        }
                    }
                }
                stage("Front") {
                    steps {
                        sh "docker-compose run --rm frontend-node-cli yarn test --watchAll=false"
                    }
                }
            }
        }
        stage("Down") {
            steps {
                sh "docker-compose down -v --remove-orphans"
            }
        }
        stage("Build") {
            steps {
                sh "docker --log-level=debug build --pull --file=gateway/docker/production/nginx/Dockerfile --tag=${REGISTRY}/auction-gateway:${IMAGE_TAG} gateway/docker"
                sh "docker --log-level=debug build --pull --file=frontend/docker/production/nginx/Dockerfile --tag=${REGISTRY}/auction-frontend:${IMAGE_TAG} frontend"
                sh "docker --log-level=debug build --pull --file=api/docker/production/nginx/Dockerfile --tag=${REGISTRY}/auction-api:${IMAGE_TAG} api"
                sh "docker --log-level=debug build --pull --file=api/docker/production/php-fpm/Dockerfile --tag=${REGISTRY}/auction-api-php-fpm:${IMAGE_TAG} api"
                sh "docker --log-level=debug build --pull --file=api/docker/production/php-cli/Dockerfile --tag=${REGISTRY}/auction-api-php-cli:${IMAGE_TAG} api"
            }
        }
        stage("Testing") {
            stages {
                stage("Build") {
                    steps {
                        sh "docker --log-level=debug build --pull --file=gateway/docker/testing/nginx/Dockerfile --tag=${REGISTRY}/auction-testing-gateway:${IMAGE_TAG} gateway/docker"
                        sh "docker --log-level=debug build --pull --file=api/docker/testing/php-cli/Dockerfile --tag=${REGISTRY}/auction-testing-api-php-cli:${IMAGE_TAG} api"
                        sh "docker --log-level=debug build --pull --file=cucumber/docker/testing/node/Dockerfile --tag=${REGISTRY}/auction-cucumber-node-cli:${IMAGE_TAG} cucumber"
                    }
                }
                stage("Init") {
                    steps {
                        sh "COMPOSE_PROJECT_NAME=testing docker-compose -f docker-compose-testing.yml up -d"
                        sh "COMPOSE_PROJECT_NAME=testing docker-compose -f docker-compose-testing.yml run --rm api-php-cli wait-for-it api-postgres:5432 -t 60"
                        sh "COMPOSE_PROJECT_NAME=testing docker-compose -f docker-compose-testing.yml run --rm api-php-cli php bin/app.php migrations:migrate --no-interaction"
                        sh "COMPOSE_PROJECT_NAME=testing docker-compose -f docker-compose-testing.yml run --rm testing-api-php-cli php bin/app.php fixtures:load --no-interaction"
                    }
                }
                stage("Smoke") {
                    steps {
                        sh "COMPOSE_PROJECT_NAME=testing docker-compose -f docker-compose-testing.yml run --rm cucumber-node-cli yarn smoke-ci"
                    }
                    post {
                        failure {
                            archiveArtifacts 'cucumber/var/*'
                        }
                    }
                }
                stage("E2E") {
                    steps {
                        sh "COMPOSE_PROJECT_NAME=testing docker-compose -f docker-compose-testing.yml run --rm cucumber-node-cli yarn e2e-ci"
                    }
                    post {
                        failure {
                            archiveArtifacts 'cucumber/var/*'
                        }
                    }
                }
                stage("Down") {
                    steps {
                        sh "COMPOSE_PROJECT_NAME=testing docker-compose -f docker-compose-testing.yml down -v --remove-orphans"
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
                sh "docker push ${REGISTRY}/auction-gateway:${IMAGE_TAG}"
                sh "docker push ${REGISTRY}/auction-frontend:${IMAGE_TAG}"
                sh "docker push ${REGISTRY}/auction-api:${IMAGE_TAG}"
                sh "docker push ${REGISTRY}/auction-api-php-fpm:${IMAGE_TAG}"
                sh "docker push ${REGISTRY}/auction-api-php-cli:${IMAGE_TAG}"
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
                        sh "make deploy"
                    }
                }
            }
        }
    }
    post {
        always {
            sh "docker-compose down -v --remove-orphans || true"
            sh "COMPOSE_PROJECT_NAME=testing docker-compose -f docker-compose-testing.yml down -v --remove-orphans || true"
            sh "rm -f docker-compose-production-env.yml || true"
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
