pipeline {
    agent any

    tools {
        maven 'Maven-3.9'
    }

    environment {
        // ── Jenkins stored credentials ──
        JFROG_USER   = credentials('jfrog-user')
        JFROG_APIKEY = credentials('jfrog-apikey')
        SONAR_TOKEN  = credentials('sonar-token')

        // ── SonarCloud — separate project for DVJA ──
        SONAR_ORG     = 'brijesh-secops'
        SONAR_PROJECT = 'brijesh-dvja'

        // ── JFrog Cloud base URL ──
        JFROG_URL = 'https://trialxcztq9.jfrog.io/artifactory'

        // ── Auto version per build ──
        APP_VERSION = "1.0.${BUILD_NUMBER}"
    }

    stages {

        // =====================================================
        // STAGE 1: CHECKOUT
        // =====================================================
        stage('Checkout') {
            steps {
                echo "📥 Checking out DVJA source code (Build #${BUILD_NUMBER})"
                checkout scm
            }
        }

        // =====================================================
        // STAGE 2: BUILD
        // =====================================================
        stage('Build') {
            steps {
                echo '🔨 Building DVJA and pulling dependencies from JFrog...'
                sh """
                    mvn clean compile \
                        -s settings.xml \
                        -Djfrog.user=${JFROG_USER} \
                        -Djfrog.apikey=${JFROG_APIKEY}
                """
            }
        }

        // =====================================================
        // STAGE 3: SONARCLOUD ANALYSIS
        // =====================================================
        stage('SonarCloud Analysis') {
            steps {
                echo '🔍 Running SonarCloud security analysis on DVJA...'
                withSonarQubeEnv('SonarCloud') {
                    sh """
                        mvn sonar:sonar \
                            -s settings.xml \
                            -Djfrog.user=${JFROG_USER} \
                            -Djfrog.apikey=${JFROG_APIKEY} \
                            -Dsonar.projectKey=${SONAR_PROJECT} \
                            -Dsonar.organization=${SONAR_ORG} \
                            -Dsonar.host.url=https://sonarcloud.io \
                            -Dsonar.token=${SONAR_TOKEN}
                    """
                }
            }
        }

        // =====================================================
        // STAGE 4: QUALITY GATE
        // =====================================================
        stage('Quality Gate') {
            steps {
                echo '🚦 Waiting for SonarCloud Quality Gate result...'
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        // =====================================================
        // STAGE 5: PACKAGE
        // =====================================================
        stage('Package') {
            steps {
                echo '📦 Packaging DVJA into WAR file...'
                sh """
                    mvn package \
                        -DskipTests \
                        -s settings.xml \
                        -Djfrog.user=${JFROG_USER} \
                        -Djfrog.apikey=${JFROG_APIKEY}
                """
                archiveArtifacts artifacts: 'target/*.war', fingerprint: true
            }
        }

        // =====================================================
        // STAGE 6: DEPLOY TO JFROG
        // =====================================================
        stage('Deploy to JFrog') {
            steps {
                echo "🚀 Deploying DVJA v${APP_VERSION} to JFrog..."
                sh """
                    mvn deploy \
                        -DskipTests \
                        -s settings.xml \
                        -Djfrog.user=${JFROG_USER} \
                        -Djfrog.apikey=${JFROG_APIKEY}
                """
            }
        }
    }

    post {
        success {
            echo """
            ✅ DVJA PIPELINE SUCCESS
            ─────────────────────────────────────
            Build      : #${BUILD_NUMBER}
            Version    : ${APP_VERSION}
            SonarCloud : https://sonarcloud.io/project/overview?id=${SONAR_PROJECT}
            JFrog      : ${JFROG_URL}/secops-pipeline-libs-snapshot-local/
            ─────────────────────────────────────
            """
        }
        failure {
            echo """
            ❌ DVJA PIPELINE FAILED
            Check SonarCloud dashboard for details:
            https://sonarcloud.io/project/overview?id=brijesh-dvja
            """
        }
        always {
            echo "Build #${BUILD_NUMBER} completed — Status: ${currentBuild.currentResult}"
            deleteDir()
        }
    }
}
