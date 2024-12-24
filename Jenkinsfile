@Library('Shared') _
pipeline {
    agent { label 'Node' }

    environment {
        SONAR_HOME = tool "Sonar"
        SONAR_AUTH_TOKEN = credentials('Sonar') // Replace 'sonar-token' with your Jenkins credential ID
    }

    parameters {
        string(name: 'FRONTEND_DOCKER_TAG', defaultValue: '', description: 'Frontend Docker tag')
        string(name: 'BACKEND_DOCKER_TAG', defaultValue: '', description: 'Backend Docker tag')
    }

    stages {
        stage("Validate Parameters") {
            steps {
                script {
                    if (params.FRONTEND_DOCKER_TAG == '' || params.BACKEND_DOCKER_TAG == '') {
                        error("FRONTEND_DOCKER_TAG and BACKEND_DOCKER_TAG must be provided.")
                    }
                }
            }
        }

        stage("Workspace Cleanup") {
            steps {
                cleanWs()
            }
        }

        stage("Git: Code Checkout") {
            steps {
                git branch: 'main',
                    url: 'https://github.com/swapnilyavalkar/Wanderlust-Mega-Project.git',
                    credentialsId: 'Github-Cred'
            }
        }

        stage("Install Dependencies") {
            steps {
                script {
                    dir("backend") {
                        sh "npm install"
                    }
                    dir("frontend") {
                        sh "npm install"
                    }
                }
            }
        }

        stage("Trivy: Filesystem Scan") {
            steps {
                sh "trivy fs ."
            }
        }

        stage("OWASP: Dependency Check") {
            steps {
                dependencyCheck odcInstallation: 'Default', additionalArguments: '--disableNodeAudit'
            }
        }

        stage("SonarQube: Code Analysis") {
            steps {
                withSonarQubeEnv('Sonar') {
                    sh """
                        ${SONAR_HOME}/bin/sonar-scanner \
                        -Dsonar.projectName=wanderlust \
                        -Dsonar.projectKey=wanderlust \
                        -Dsonar.login=${env.SONAR_AUTH_TOKEN}
                    """
                }
            }
        }

        stage("SonarQube: Code Quality Gates") {
            steps {
                timeout(time: 1, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage("Exporting Environment Variables") {
            parallel {
                stage("Backend Env Setup") {
                    steps {
                        dir("Automations") {
                            sh "bash updatebackendnew.sh"
                        }
                    }
                }
                stage("Frontend Env Setup") {
                    steps {
                        dir("Automations") {
                            sh "bash updatefrontendnew.sh"
                        }
                    }
                }
            }
        }

        stage("Docker: Build Images") {
            steps {
                sh """
                    docker build -t swapnilyavalkar/wanderlust-backend-beta:${params.BACKEND_DOCKER_TAG} backend/
                    docker build -t swapnilyavalkar/wanderlust-frontend-beta:${params.FRONTEND_DOCKER_TAG} frontend/
                """
            }
        }

        stage("Docker: Push to DockerHub") {
            steps {
                sh """
                    docker push swapnilyavalkar/wanderlust-backend-beta:${params.BACKEND_DOCKER_TAG}
                    docker push swapnilyavalkar/wanderlust-frontend-beta:${params.FRONTEND_DOCKER_TAG}
                """
            }
        }
    }

    post {
        always {
            cleanWs()
        }
        success {
            archiveArtifacts artifacts: '*.xml', followSymlinks: false
            build job: "Wanderlust-CD", parameters: [
                string(name: 'FRONTEND_DOCKER_TAG', value: "${params.FRONTEND_DOCKER_TAG}"),
                string(name: 'BACKEND_DOCKER_TAG', value: "${params.BACKEND_DOCKER_TAG}")
            ]
        }
        failure {
            echo 'Pipeline failed.'
        }
    }
}
