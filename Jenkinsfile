pipeline {
    agent any

    tools {
        maven 'Maven'
    }

    environment {
        APP_REPO   = 'https://github.com/rahuldutta0487/maven-web-application.git'
        APP_BRANCH = 'master'
        SONAR_KEY  = 'maven-web-app'
        WAR_NAME   = 'maven-web-app.war'
    }

    options {
        timestamps()
        buildDiscarder(logRotator(numToKeepStr: '10'))
    }

    stages {

        stage('1. Checkout') {
            steps {
                echo "Cloning ${APP_REPO} (${APP_BRANCH})"
                git branch: "${APP_BRANCH}", url: "${APP_REPO}"
                sh 'git log -1 --oneline'
            }
        }

        stage('2. SonarQube Analysis') {
            steps {
                withCredentials([string(credentialsId: 'sonar-token', variable: 'SONAR_TOKEN')]) {
                    sh '''
                        mvn clean compile sonar:sonar \
                        -Dsonar.projectKey=maven-web-app \
                        -Dsonar.projectName=maven-web-app \
                        -Dsonar.host.url=http://13.207.2.177:9000 \
                        -Dsonar.login=squ_10d5f5237c553c596ef7951cdd0cc5e5fe654121
                    '''
                }
            }
        }
        

        stage('3. Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('4. Build WAR') {
            steps {
                sh 'mvn clean package -DskipTests'
                sh 'ls -lh target/*.war'
                sh "cp target/*.war ${WORKSPACE}/${WAR_NAME}"
            }
            post {
                success {
                    archiveArtifacts artifacts: "${WAR_NAME}", fingerprint: true
                }
            }
        }

        stage('5. Deploy via Ansible') {
            steps {
                sh """
                    ansible-playbook -i hosts.txt deploy.yml \
                    --extra-vars "war_src=${WORKSPACE}/${WAR_NAME} war_name=${WAR_NAME}"
                """
            }
        }
    }

    post {
        success {
            echo "✅ Pipeline SUCCESS — deployed ${WAR_NAME} to Tomcat."
        }
        failure {
            echo "❌ Pipeline FAILED — Tomcat was NOT updated. Check logs."
        }
        always {
            echo "Build #${env.BUILD_NUMBER} finished with status: ${currentBuild.currentResult}"
        }
    }
}
