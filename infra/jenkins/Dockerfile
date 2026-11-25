FROM jenkins/jenkins:lts-jdk21
USER root
# CLI de Docker + plugin compose para que los pipelines construyan y desplieguen
RUN apt-get update && apt-get install -y --no-install-recommends ca-certificates curl \
 && install -m 0755 -d /etc/apt/keyrings \
 && curl -fsSL https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc \
 && echo "deb [signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/debian $(. /etc/os-release && echo $VERSION_CODENAME) stable" > /etc/apt/sources.list.d/docker.list \
 && apt-get update && apt-get install -y --no-install-recommends docker-ce-cli docker-compose-plugin \
 && rm -rf /var/lib/apt/lists/*
# Plugins mínimos para el pipeline
RUN jenkins-plugin-cli --plugins workflow-aggregator git docker-workflow htmlpublisher

# OWASP ZAP para pruebas dinámicas de seguridad (DAST); usa el Java que ya trae Jenkins
ARG ZAP_VERSION=2.17.0
RUN curl -sSL "https://github.com/zaproxy/zaproxy/releases/download/v${ZAP_VERSION}/ZAP_${ZAP_VERSION}_Linux.tar.gz" \
    | tar -xz -C /opt \
 && ln -s /opt/ZAP_${ZAP_VERSION}/zap.sh /usr/local/bin/zap.sh

# Laboratorio local: root para poder usar el socket de Docker
USER root
