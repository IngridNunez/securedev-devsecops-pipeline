// Pipeline CI/CD - SecureDev Tickets (OCY1102 Actividad 3)
// Etapas actuales: Checkout -> Build -> Test -> Deploy.
// Las etapas de seguridad (pip-audit, bandit, OWASP ZAP) se agregan en commits posteriores.

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
