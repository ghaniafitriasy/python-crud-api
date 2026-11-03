FROM docker:cli AS dockercli

FROM jenkins/jenkins:lts-jdk17
USER root
COPY --from=dockercli /usr/local/bin/docker /usr/local/bin/docker
