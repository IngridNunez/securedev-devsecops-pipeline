// Pipeline CI/CD - SecureDev Tickets (OCY1102 Actividad 3)
// Etapas: Checkout -> Build -> Test -> DAST (OWASP ZAP) -> Security Scan (bandit/pip-audit) -> Deploy.
// DAST documenta hallazgos sin bloquear (unstable); Security Scan sí bloquea el Deploy (error).

pipeline {
    agent any

    options {
        timeout(time: 20, unit: 'MINUTES')
        buildDiscarder(logRotator(numToKeepStr: '20'))
        disableConcurrentBuilds()
    }

    environment {
        APP_DIR      = 'securedev-lab'
        IMAGE        = 'securedev-app'
        // Mismo nombre de proyecto que la app levantada a mano, para reemplazar sus contenedores
        // y mantener la red que Prometheus usa para recolectar métricas.
        COMPOSE_PROJ = 'securedev-estudiante'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
                sh 'git log -1 --pretty="Commit: %h | Autor: %an | %s"'
            }
        }

        stage('Build') {
            steps {
                sh 'docker build -t ${IMAGE}:${BUILD_NUMBER} ${APP_DIR}'
            }
        }

        stage('Test') {
            steps {
                // Prueba de humo dentro de la imagen recién construida:
                // /health responde 200 y las rutas protegidas redirigen al login sin sesión.
                sh '''
                    docker run --rm -e DATABASE_PATH=/tmp/test.db --entrypoint python ${IMAGE}:${BUILD_NUMBER} -c "
from app import create_app
c = create_app().test_client()
assert c.get('/health').status_code == 200, 'health no responde 200'
assert c.get('/').status_code == 302, 'la portada debe redirigir al login'
assert c.get('/tickets/1').status_code == 302, 'ticket sin sesion debe redirigir'
print('Prueba de humo OK')
"
                '''
            }
        }

        stage('DAST (OWASP ZAP)') {
            steps {
                // Pruebas dinámicas de seguridad: ZAP ataca la app ya desplegada
                // (http://app:8000, misma red Docker) con el plan de zap.yaml.
                // No bloquea el Deploy (a diferencia de Security Scan): solo
                // documenta los hallazgos y marca la build como inestable si
                // encuentra alertas de riesgo medio o alto.
                script {
                    sh 'mkdir -p zapreports'
                    def rc = sh(script: 'zap.sh -cmd -port 8091 -autorun "$WORKSPACE/zap.yaml"', returnStatus: true)
                    archiveArtifacts artifacts: 'zapreports/*', allowEmptyArchive: true
                    if (rc == 1) {
                        unstable('DAST (ZAP): alertas de riesgo ALTO encontradas. Ver zapreports/zap-report.html')
                    } else if (rc == 2) {
                        unstable('DAST (ZAP): alertas de riesgo MEDIO encontradas. Ver zapreports/zap-report.html')
                    }
                }
            }
        }

        stage('Security Scan') {
            steps {
                // Puerta de seguridad (quality gate): si bandit o pip-audit encuentran
                // vulnerabilidades, la etapa falla y el Deploy NO se ejecuta.
                //  - bandit: análisis estático del código Python (SAST)
                //  - pip-audit: dependencias con vulnerabilidades conocidas (CVE/PYSEC)
                // Se ejecuta en un contenedor aparte; los archivos se copian con docker cp
                // porque Jenkins corre en un contenedor y no comparte rutas con el host.
                script {
                    // Siempre se imprime y archiva el informe; el código de salida decide el resultado.
                    def rc = sh(returnStatus: true, script: '''
                        rm -rf reports && mkdir -p reports
                        cid=$(docker create python:3.11-slim sh -c "mkdir -p /out && pip install -q --disable-pip-version-check bandit pip-audit >/dev/null 2>&1; cd /src; bandit -r app.py -f txt -o /out/bandit.txt; echo \\$? > /out/bandit.rc; pip-audit -r requirements.txt > /out/pip-audit.txt 2>&1; echo \\$? > /out/pip-audit.rc; true")
                        docker cp ${APP_DIR}/. $cid:/src
                        docker start -a $cid > /dev/null 2>&1 || true
                        docker cp $cid:/out/. reports/
                        docker rm $cid > /dev/null
                        echo "=========== BANDIT (análisis estático) ==========="
                        cat reports/bandit.txt
                        echo "=========== PIP-AUDIT (dependencias) ==========="
                        cat reports/pip-audit.txt
                        # Falla si cualquiera de las dos herramientas encontró algo (código de salida distinto de 0)
                        [ "$(cat reports/bandit.rc)" = "0" ] && [ "$(cat reports/pip-audit.rc)" = "0" ]
                    ''')
                    archiveArtifacts artifacts: 'reports/*', allowEmptyArchive: true
                    if (rc != 0) {
                        error('Security Scan: se encontraron vulnerabilidades. Despliegue bloqueado hasta corregirlas.')
                    }
                }
            }
        }

        stage('Deploy') {
            steps {
                // Despliega exactamente la imagen construida y probada en las etapas anteriores:
                // el override fija la etiqueta y --no-build impide reconstruirla.
                sh '''
                    printf 'services:
  app:
    image: %s:%s
' "${IMAGE}" "${BUILD_NUMBER}" > deploy.override.yml
                    docker compose -p ${COMPOSE_PROJ} -f ${APP_DIR}/compose.yaml -f deploy.override.yml up -d --no-build
                '''
                // Espera a que la app responda (hasta ~40 s)
                sh '''
                    for i in $(seq 1 20); do
                        if docker compose -p ${COMPOSE_PROJ} -f ${APP_DIR}/compose.yaml -f deploy.override.yml exec -T app python -c "import urllib.request,sys; sys.exit(0 if urllib.request.urlopen('http://localhost:8000/health').status == 200 else 1)"; then
                            echo "Despliegue OK: ${IMAGE}:${BUILD_NUMBER} responde en /health"
                            exit 0
                        fi
                        sleep 2
                    done
                    echo "La aplicación no respondió tras el despliegue"
                    exit 1
                '''
            }
        }
    }

    post {
        success { echo "Pipeline OK - imagen ${IMAGE}:${BUILD_NUMBER}" }
        failure { echo 'Pipeline FALLÓ - revisar el log de la etapa en rojo' }
    }
}
