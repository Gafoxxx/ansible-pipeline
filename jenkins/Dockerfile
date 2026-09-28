# Equivalente a madmilodz/jenkinsci:v3 sobre el Jenkins LTS actual
# (los plugins "latest" ya no son compatibles con Jenkins 2.387.2)
FROM jenkins/jenkins:lts-jdk21

USER root
RUN apt-get update \
    && apt-get install -y --no-install-recommends python3 python3-pip python-is-python3 nodejs npm sshpass \
    && rm -rf /var/lib/apt/lists/*

COPY plugins.txt /plugins.txt
RUN jenkins-plugin-cli --plugin-file /plugins.txt

# Usuario admin y seguridad (usa JENKINS_ADMIN_ID / JENKINS_ADMIN_PASSWORD)
COPY init.groovy.d/ /usr/share/jenkins/ref/init.groovy.d/
