pipeline {
  agent any

  tools { 
    jdk 'jdk-17'           // En Jenkins: nombre de la instalación JDK
    maven 'maven-3.9'      // En Jenkins: nombre de la instalación Maven
  }

  options { 
    timestamps()
    ansiColor('xterm') 
    disableConcurrentBuilds()
  }

  environment {
    // Jenkins (en contenedor) se conecta al host por host.docker.internal
    STAGING_HOST = 'host.docker.internal'
    STAGING_PORT = '2222'                 // Puerto SSH del host/VM
    APP_DIR      = '/opt/springboot/app'
    APP_PORT     = '8080'                 // Puerto donde corre la app
  }

  stages {
    stage('Checkout') {
      steps { checkout scm }
    }

    stage('Build & Unit Tests') {
      steps { sh 'mvn -B -U clean test' }
      post { always { junit '**/target/surefire-reports/*.xml' } }
    }

    stage('Checkstyle') {
      steps { sh 'mvn -B -DskipTests checkstyle:check' }
    }

    stage('Coverage (JaCoCo)') {
      steps { sh 'mvn -B -DskipTests jacoco:report' }
      post {
        success {
          jacoco(
            execPattern: '**/target/jacoco.exec',
            classPattern: '**/target/classes',
            sourcePattern: '**/src/main/java'
          )
        }
      }
    }

    stage('Package') {
      steps {
        sh '''
          set -euo pipefail
          mvn -B -DskipTests package
          ls -1 target/*.jar | head -n1 > JAR_PATH.txt
        '''
        archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
      }
    }

    stage('Deploy to Staging (SSH)') {
      steps {
        sshagent (credentials: ['staging-ssh']) {
          sh '''
            set -euxo pipefail
            JAR=$(cat JAR_PATH.txt)
            BASENAME=$(basename "$JAR")

            # Crear directorio remoto
            ssh -p ${STAGING_PORT} -o StrictHostKeyChecking=no deploy@${STAGING_HOST} "mkdir -p ${APP_DIR}"

            # Copiar jar al servidor
            scp -P ${STAGING_PORT} -o StrictHostKeyChecking=no "$JAR" deploy@${STAGING_HOST}:${APP_DIR}/

            # Parar instancia previa (si existe) y arrancar nueva
            ssh -p ${STAGING_PORT} -o StrictHostKeyChecking=no deploy@${STAGING_HOST} "\
              pkill -f ${APP_DIR}/springboot-app.jar || true; \
              ln -sfn ${APP_DIR}/${BASENAME} ${APP_DIR}/springboot-app.jar; \
              nohup java -jar ${APP_DIR}/springboot-app.jar --server.port=${APP_PORT} > ${APP_DIR}/app.log 2>&1 & echo \$! > ${APP_DIR}/app.pid"
          '''
        }
      }
    }

    stage('Health Check') {
      steps {
        sh '''
          set -euxo pipefail
          URL="http://${STAGING_HOST}:${APP_PORT}/actuator/health"
          for i in $(seq 1 10); do
            sleep 3
            if curl -fsS "$URL" | grep -q '"status":"UP"'; then
              echo "Service UP en $URL"
              exit 0
            fi
          done
          echo "Fallo health check"; curl -i "$URL" || true; exit 1
        '''
      }
    }
  }

  post {
    always {
      echo 'Pipeline finalizado.'
    }
  }
}
