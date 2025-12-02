pipeline {
  agent any

  environment {
    MAVEN_TOOL    = 'Maven3'
    REMOTE_USER   = 'jenkins-deploy'
    REMOTE_HOST   = '192.168.122.2'
    SSH_CRED      = 'jenkins-deploy-ssh'
    REMOTE_TMP    = '/tmp'
    DEPLOY_SCRIPT = '/usr/local/bin/deploy_war.sh'
    WAR_GLOB      = 'target/*.war'
  }

  options {
    timestamps()
    timeout(time: 30, unit: 'MINUTES')
    buildDiscarder(logRotator(numToKeepStr: '10'))
  }

  stages {

    stage('Checkout') {
      steps { checkout scm }
    }

    stage('Build') {
      tools { maven "${env.MAVEN_TOOL}" }
      steps {
        sh '''
          set -euo pipefail
          mvn -B -DskipTests=false clean package
        '''
        archiveArtifacts artifacts: "${WAR_GLOB}", fingerprint: true
      }
    }

    stage('Unit Tests (publish)') {
      steps {
        junit allowEmptyResults: true, testResults: '**/target/surefire-reports/*.xml'
      }
    }

    stage('Deploy (secure)') {
      steps {
        sshagent (credentials: ["${SSH_CRED}"]) {
          sh '''
            set -euo pipefail
            NEWWAR=app_${BUILD_NUMBER}.war
            scp -o StrictHostKeyChecking=no ${WAR_GLOB} ${REMOTE_USER}@${REMOTE_HOST}:${REMOTE_TMP}/${NEWWAR}
            ssh -o StrictHostKeyChecking=no ${REMOTE_USER}@${REMOTE_HOST} "sudo ${DEPLOY_SCRIPT} ${REMOTE_TMP}/${NEWWAR}"
          '''
        }
      }
    }

    stage('Collect Tomcat Status & Logs') {
      steps {
        sshagent (credentials: ["${SSH_CRED}"]) {
          sh '''
            set -euo pipefail
            echo "=== REMOTE: Tomcat status & last logs ==="
            ssh -o StrictHostKeyChecking=no ${REMOTE_USER}@${REMOTE_HOST} "sudo /usr/local/bin/check_tomcat.sh" || true
          '''
        }
      }
    }

    stage('Smoke Test') {
      steps {
        script {
          retry(3) {
            sleep 5
            def code = sh(script: "curl -s -o /dev/null -w '%{http_code}' http://${REMOTE_HOST}:8080/", returnStdout: true).trim()
            if (code != '200') {
              error("Smoke test failed: HTTP ${code}")
            }
            echo "Smoke test OK: HTTP ${code}"
          }
        }
      }
    }

    stage('Optional: Auto-Rollback on Failure') {
      when { expression { currentBuild.currentResult == 'FAILURE' } }
      steps {
        sshagent (credentials: ["${SSH_CRED}"]) {
          sh '''
            ssh -o StrictHostKeyChecking=no ${REMOTE_USER}@${REMOTE_HOST} \
            "sudo /usr/local/bin/rollback_latest_tomcat.sh || echo 'rollback failed/none found'"
          '''
        }
      }
    }

  }

  post {
    success { echo "Deployment succeeded: ${env.BUILD_URL}" }
    failure { echo "Pipeline failed — check logs and Tomcat output." }
    always { cleanWs() }
  }
}
