// =====================================================================
// Enterprise CI/CD Pipeline — Java Maven Web Application (Milestone 3)
// Jenkins (Declarative) | SonarQube Quality Gate | Maven WAR | Ansible -> Tomcat
//
// REUSES the same Jenkins / SonarQube / Tomcat / Ansible from the e-commerce
// case study. Only the repo URL, Sonar project key and WAR name change.
// =====================================================================

pipeline {
    agent any

    // Tools already configured globally in Manage Jenkins -> Tools (reused)
    tools {
        maven 'maven3'   // <-- name MUST match your Jenkins Maven tool name
        jdk   'jdk17'    // <-- name MUST match your Jenkins JDK tool name (or remove if using system JDK)
    }

    environment {
        // ---- EDIT THESE 4 VALUES ----
        APP_REPO     = 'https://github.com/rahuldutta0487/maven-web-application.git'  // instructor's Maven repo
        APP_BRANCH   = 'main'                       // or 'master'
        SONAR_KEY    = 'maven-web-app'              // new Sonar project (token is reused)
        WAR_NAME     = 'maven-web-app.war'          // final WAR name copied to Tomcat
        // -----------------------------

        SONAR_SERVER = 'sonar-server'               // name of SonarQube server in Jenkins (reused)
    }

    options {
        timestamps()
        buildDiscarder(logRotator(numToKeepStr: '10'))
    }

    stages {

        // ---------- STAGE 1: Source Code Management ----------
        stage('1. Checkout') {
            steps {
                echo "Cloning ${APP_REPO} (${APP_BRANCH})"
                git branch: "${APP_BRANCH}", url: "${APP_REPO}"
                sh 'git log -1 --oneline'   // version tracking
            }
        }

        // ---------- STAGE 2: Static Code Analysis ----------
        stage('2. SonarQube Analysis') {
            steps {
                // 'sonar-server' + 'sonar-token' credential reused from e-commerce setup
                withSonarQubeEnv("${SONAR_SERVER}") {
                    sh """
                        mvn clean compile sonar:sonar \
                          -Dsonar.projectKey=${SONAR_KEY} \
                          -Dsonar.projectName=${SONAR_KEY}
                    """
                }
            }
        }

        // ---------- STAGE 3: Quality Gate Decision ----------
        stage('3. Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    // Pipeline ABORTS automatically if gate = FAIL  (abortPipeline: true)
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        // ---------- STAGE 4: Application Build (WAR) ----------
        stage('4. Build WAR') {
            steps {
                sh 'mvn clean package -DskipTests'
                sh 'ls -lh target/*.war'
                // Normalise the artifact name so deploy is predictable
                sh "cp target/*.war ${WORKSPACE}/${WAR_NAME}"
            }
            post {
                success {
                    archiveArtifacts artifacts: "${WAR_NAME}", fingerprint: true
                }
            }
        }

        // ---------- STAGE 5: Deployment Automation (Ansible -> Tomcat) ----------
        stage('5. Deploy via Ansible') {
            steps {
                // deploy.yml: stop Tomcat -> remove old WAR -> copy new WAR -> start -> verify
                sh """
                    ansible-playbook -i hosts.txt deploy.yml \
                      --extra-vars "war_src=${WORKSPACE}/${WAR_NAME} war_name=${WAR_NAME}"
                """
            }
        }
    }

    // ---------- Post / Notifications / Failure handling ----------
    post {
        success {
            echo "✅ Pipeline SUCCESS — deployed ${WAR_NAME} to Tomcat."
        }
        failure {
            echo "❌ Pipeline FAILED — Tomcat was NOT updated (previous version still serving). Check logs."
        }
        always {
            echo "Build #${env.BUILD_NUMBER} finished with status: ${currentBuild.currentResult}"
        }
    }
}
