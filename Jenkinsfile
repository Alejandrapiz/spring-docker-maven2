pipeline {
  agent any
  tools { jdk 'jdk-17'; maven 'maven-3.9' }
  options { timestamps(); ansiColor('xterm'); disableConcurrentBuilds() }

  environment {
    // Si accedes por localhost:2222 (puerto publicado desde tu contenedor/VM):
    STAGING_HOST = 'localhost'
    STAGING_PORT = '2222'

    // Si Jenkins y Staging están en la MISMA red Docker (sin publicar 2222):
    // STAGING_HOST = 'maven1'
    // STAGING_PORT = '22'

    APP_DIR = '/opt/springboot/app'
  }

  stages {
    stage('Checkout') {
      steps { checkout scm }
    }

    stage('Build & Unit Tests') {
      steps { sh 'mvn -B -U clean verify' }           // compila + tests
      post { always { junit '**/target/surefire-reports/*.xml' } }
    }

    stage('Checkstyle') {
      steps { sh 'mvn -B checkstyle:check' }          // formatea/valida estilo
    }

    stage('Coverage (JaCoCo)') {
      steps { sh 'mvn -B jacoco:report' }             // genera reporte
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
        sh 'ls -1 target/*.jar | head -n1 > JAR_PATH.txt'
        archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
      }
    }

    stage('Deploy to Staging (SSH)') {
      steps {
        sshagent (credentials: ['staging-ssh']) {     // crea esta credencial en Jenkins
          sh '''
            set -euxo pipefail
            JAR=$(cat JAR_PATH.txt)

            # crear directorio remoto
            ssh -p ${STAGING_PORT} -o StrictHostKeyChecking=no deploy@${STAGING_HOST} "mkdir -p ${APP_DIR}"

            # copiar jar
            scp -P ${STAGING_PORT} -o StrictHostKeyChecking=no "$JAR" deploy@${STAGING_HOST}:${APP_DIR}/

            # arrancar con nohup y dejar symlink estable springboot-app.jar
            ssh -p ${STAGING_PORT} -o StrictHostKeyChecking=no deploy@${STAGING_HOST} "\
              pkill -f springboot-app.jar || true; \
              nohup java -jar ${APP_DIR}/$(basename "$JAR") > /dev/null 2>&1 & \
              ln -sfn ${APP_DIR}/$(basename "$JAR") ${APP_DIR}/springboot-app.jar"
          '''
        }
      }
    }

    stage('Health Check') {
      steps {
        sh '''
          set -euxo pipefail
          # Si usas localhost con puerto 8080 publicado:
          URL="http://localhost:8080/actuator/health"
          # Si usas red interna y el host se llama maven1, usa:
          # URL="http://maven1:8080/actuator/health"

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
}

