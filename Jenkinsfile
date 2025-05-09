import groovy.json.JsonOutput

boolean isContainerConnected(network, container) {
    def result = sh(script: "docker network inspect ${network} --format '{{range .Containers}}{{if eq .Name \"${container}\"}}connected{{end}}{{end}}'", returnStdout: true).trim()
    return result == 'connected'
}
pipeline {
    agent any

    environment {

        // API_URL = credentials('api_url') // NewMan allure-reports
        GITHUB_TOKEN = credentials('github_token')
        REACT_APP_API_URL_TEST = credentials('react_app_api_url_test_env')
        DOMAIN_NAME = credentials('domain_name')
        //GIT_COMMIT_MAIN_REPO = credentials('git_commit_main_repo')
        DB_URL = credentials('test_db_url')//
        DB_NAME = credentials('db_name_shop_test_env')
        DB_USER = credentials('db_user')
        DB_PASSWORD = credentials('db_password_shop_test_env')
        JWT_SECRET = credentials('jwt_secret')
        JWT_REFRESH_SECRET = credentials('jwt_refresh_secret')
        CORS_ALLOWED_ORIGINS = credentials('cors_allowed_origins_test_env')
    }

    stages {
        stage('Stop Old Containers') {
            steps {
                script {
                    def containers = ["test_env_frontend_1", "test_env_backend_1"]
                    containers.each { container ->
                        sh """
                            if [ \$(docker ps -a -q -f name=${container}) ]; then
                                docker stop ${container} || true
                                docker rm ${container} || true
                            else
                                echo "Container ${container} does not exist, skipping..."
                            fi
                        """
                    } 
                    // sh 'docker-compose -f docker-compose.dev.yml down'
                }
            }
        }

        stage('Create .env File') {
            steps {
                script {
                    echo 'Creating .env file...'
                    
                    def envContent = """
                    REACT_APP_API_URL_TEST=${REACT_APP_API_URL_TEST}
                    DOMAIN_NAME=app.${DOMAIN_NAME}
                    DB_NAME=${DB_NAME}
                    DB_URL=${DB_URL}
                    DB_USER=${DB_USER}
                    DB_PASSWORD=${DB_PASSWORD}
                    JWT_SECRET=${JWT_SECRET}
                    JWT_REFRESH_SECRET=${JWT_REFRESH_SECRET}
                    CORS_ALLOWED_ORIGINS=${CORS_ALLOWED_ORIGINS}
                    """

                    writeFile file: './frontend/webform/.env', text: envContent.stripIndent() //writeFile file: './frontend/webform/.env'
                }
            }
        }
        
        stage('Build and Run') {
            steps {
                script {
                    echo 'Deploy to Test Server'
                    echo 'Using updated docker-compose for test environment!'
                    echo 'Starting Docker Compose...'
                    sh 'docker --version'
                    sh 'docker-compose --version'
                    sh 'docker-compose --env-file ./frontend/webform/.env -f docker-compose.dev.yml up -d --build'
                    if (!isContainerConnected('test_env_test-net', 'server-webserver-1')) {
                    sh 'docker network connect test_env_test-net server-webserver-1'}
                    if (!isContainerConnected('test_env_test-net', 'wg-easy')) {
                    sh 'docker network connect test_env_test-net wg-easy'  }
                }
                
            }
        }

        stage('Run Tests') {
            steps {
                script {

                    echo 'Example of tests in Test Env.'
                    echo 'Running tests with Newman...'
                    sh 'newman -v'
                    // echo "Using API URL: " //$API_URL
                    // 
                    // Запуск тестов с использованием API_URL
                    // sh ('newman run $API_URL --reporters cli,allure --reporter-allure-export ./allure-results-frontend')

                    //error("Force failure for testing purposes")
                }
            }
        }

        stage('Generate Allure Report') {
            steps {
                script {
                    echo 'Generating Allure report...'
                    sh 'allure --version'
                    sh 'allure generate ./allure-results-frontend -o ./allure-report-frontend --clean'
                }
            }
        }

        stage('Publish Allure Report') {
            steps {
                script {
                    echo 'Publishing Allure report...'
                    allure([
                        results: [[path: './allure-results-frontend']]
                    ])
                }
            }
        }
    }

    post {
        always {
            script {
                // echo 'Cleaning up...'
                // sh 'docker-compose down'
                echo 'Trying to save current version available for testers.'
                echo 'Test deliverables will help to decide when to make Pull Request to main branch.'
            }
        }

        success {
            script {
                echo 'Build succeeded!'
                def commitStatusUrl = "https://api.github.com/repos/ADovhal/WebShopOnline/statuses/${env.GIT_COMMIT}"

                def body = JsonOutput.toJson([
                    state: 'success',
                    target_url: env.BUILD_URL,
                    description: "Build succeeded",
                    context: "continuous-integration/jenkins"
                ])

                writeFile file: 'body.json', text: body

                sh("""
                    curl -X POST -H "Authorization: token \$GITHUB_TOKEN" -H "Content-Type: application/json" \
                    -d @body.json ${commitStatusUrl}
                """)
            }
        }

        failure {
            script {
                echo 'Build failed!'
              
                def commitStatusUrl = "https://api.github.com/repos/ADovhal/WebShopOnline/statuses/${env.GIT_COMMIT}"

                def body = JsonOutput.toJson([
                    state: 'failure',
                    target_url: env.BUILD_URL,
                    description: "Build failed",
                    context: "continuous-integration/jenkins"
                ])

                writeFile file: 'body.json', text: body

                sh("""
                    curl -X POST -H "Authorization: token \$GITHUB_TOKEN" -H "Content-Type: application/json" \
                    -d @body.json ${commitStatusUrl}
                """)
            }
        }
    }
}


