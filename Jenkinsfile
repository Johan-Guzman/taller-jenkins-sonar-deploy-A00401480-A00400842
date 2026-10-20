pipeline {
    agent any

    environment {
        // Nombre del servidor SonarQube configurado en Administrar Jenkins -> System
        SONAR_SERVER_NAME = 'SonarQube-Server'
        PROJECT_KEY       = 'product-api'
        IMAGE_NAME        = 'product-api'
        APP_PORT          = '8085'
    }

    stages {
        // =========================================================================
        // ETAPA 1: PRUEBAS Y COBERTURA DE CODIGO (JACOCO)
        // =========================================================================
        stage('Pruebas & Cobertura JaCoCo') {
            steps {
                echo '===> [ETAPA 1] Compilando y ejecutando pruebas unitarias e instrumentacion JaCoCo...'
                dir('backend') {
                    sh 'mvn clean verify'
                }
            }
            post {
                always {
                    // Publica reporte visual de pruebas JUnit en la interfaz de Jenkins
                    junit '**/target/surefire-reports/*.xml'
                }
            }
        }

        // =========================================================================
        // ETAPA 2: VALIDACION SAST (SONARQUBE) Y BLOQUEO POR QUALITY GATE
        // =========================================================================
        stage('SonarQube SAST Scan') {
            steps {
                echo '===> [ETAPA 2] Ejecutando analisis estatico SAST con SonarQube...'
                dir('backend') {
                    withSonarQubeEnv("${SONAR_SERVER_NAME}") {
                        sh '''
                            mvn sonar:sonar \
                              -Dsonar.projectKey=${PROJECT_KEY} \
                              -Dsonar.coverage.jacoco.xmlReportPaths=target/site/jacoco/jacoco.xml
                        '''
                    }
                }
            }
        }

        stage('Quality Gate') {
            steps {
                echo '===> [ETAPA 2] Esperando veredicto reactivo del Quality Gate via Webhook...'
                timeout(time: 5, unit: 'MINUTES') {
                    script {
                        def qg = waitForQualityGate()
                        if (qg.status != 'OK') {
                            error "PIPELINE ABORTADO: Quality Gate fallo con veredicto: ${qg.status}. Corrija defectos y cobertura."
                        }
                        echo "Quality Gate APROBADO exitosamente (${qg.status}). Cumple estandares de confiabilidad y seguridad."
                    }
                }
            }
        }

        // =========================================================================
        // ETAPA 3: PUBLICACION DE ARTEFACTOS INMUTABLES (NEXUS + DOCKER)
        // =========================================================================
        

stage('Publicacion en Nexus') {
    steps {
        echo '===> [ETAPA 3] Publicando artefacto Maven en Nexus...'

        withCredentials([
            usernamePassword(
                credentialsId: 'nexus-maven-credentials',
                usernameVariable: 'NEXUS_USER',
                passwordVariable: 'NEXUS_PASS'
            )
        ]) {
            dir('backend') {
                sh '''
                    mvn -B -s settings.xml deploy -DskipTests
                '''
            }
        }
    }
}



        stage('Empaquetar Imagen Docker') {
        
            steps {
                echo "===> [ETAPA 3] Construyendo imagen de contenedor Docker: ${IMAGE_NAME}:${BUILD_NUMBER}..."
                sh "docker build -t ${IMAGE_NAME}:${BUILD_NUMBER} -t ${IMAGE_NAME}:latest backend/"
            }
        }

        // =========================================================================
        // ETAPA 4: DESPLIEGUE CONTINUO (CD) Y SMOKE TEST / HEALTHCHECK
        // =========================================================================
      

stage('Despliegue Continuo (CD)') {
    steps {
        echo "===> [ETAPA 4] Desplegando ${IMAGE_NAME}:${BUILD_NUMBER}..."

        sh """
            docker stop product-api-running || true
            docker rm product-api-running || true

            docker run -d \
                --name product-api-running \
                --network cicl_network \
                -p ${APP_PORT}:8080 \
                ${IMAGE_NAME}:${BUILD_NUMBER}

            echo '=== ESTADO DEL CONTENEDOR ==='
            docker ps -a --filter name=product-api-running
        """

        echo '===> [ETAPA 4] Smoke Test y diagnóstico'

        sh '''
            i=1

            while [ "$i" -le 10 ]; do
                echo "=== INTENTO $i DE 10 ==="

                if curl -s -f --max-time 5 \
                    http://product-api-running:8080/api/products \
                    >/dev/null 2>&1; then
                    echo "DESPLIEGUE EXITOSO: la API responde."
                    exit 0
                fi

                echo "La API todavía no responde."
                sleep 3
                i=$((i + 1))
            done

            echo "========== DIAGNOSTICO DEL DESPLIEGUE =========="

            echo "=== 1. ESTADO DEL CONTENEDOR ==="
            docker ps -a --filter name=product-api-running || true

            echo "=== 2. LOGS DE SPRING BOOT ==="
            docker logs --timestamps --tail 200 product-api-running 2>&1 || true

            echo "=== 3. INSPECCION DEL CONTENEDOR ==="
            docker inspect product-api-running \
                --format 'Estado={{.State.Status}} ExitCode={{.State.ExitCode}} Error={{.State.Error}}' \
                2>&1 || true

            echo "=== 4. PRUEBA HTTP DESDE JENKINS ==="
            curl -v --max-time 10 \
                http://product-api-running:8080/api/products \
                2>&1 || true

            echo "========== FIN DEL DIAGNOSTICO =========="
            exit 1
        '''
    }
}


    }

    post {
        success {
            echo '========================================================================='
            echo '🎉 PIPELINE EXITOSO: Integracion, Calidad, Publicacion y Despliegue OK!'
            echo '========================================================================='
        }
        failure {
            echo '========================================================================='
            echo '❌ PIPELINE FALLIDO: Revise la consola del stage que provoco el error.'
            echo '========================================================================='
        }
    }
}
