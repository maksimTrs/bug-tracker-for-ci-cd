# Jenkins LTS with JDK 17, pinned for build determinism.
# To upgrade: check https://www.jenkins.io/changelog-stable/ and update the tag.
# For maximum reproducibility pin by digest: jenkins/jenkins@sha256:<digest>.
FROM jenkins/jenkins:2.479.3-lts-jdk17

USER root

# Why Docker CLI?
# Jenkins runs as a Docker container (see docker-compose.yml), but Jenkinsfile pipeline
# steps need to execute Docker commands (docker build, docker run, docker push, etc.).
# Installing only the CLI binary gives pipelines those commands — without running a daemon.
# The actual Docker daemon runs in a separate sidecar container (jenkins-docker, DinD).
# Three env vars in docker-compose.yml wire the CLI to that daemon over TCP/TLS:
#   DOCKER_HOST        — address of the DinD daemon (tcp://docker:2376)
#   DOCKER_CERT_PATH   — path to client TLS cert+key (mounted from shared certs volume)
#   DOCKER_TLS_VERIFY  — enforce mutual TLS (daemon rejects unverified clients)
#
# Package breakdown:
#   ca-certificates + curl  — fetch Docker's apt GPG signing key over HTTPS
#   lsb-release             — provides distro codename ("bookworm") for the apt repo URL
#   docker-ce-cli           — Community Edition CLI only; no daemon, no containerd
#
# Single RUN layer + --no-install-recommends + apt cache cleanup in the same layer = lean image.
RUN apt-get update \
 && apt-get install -y --no-install-recommends \
      ca-certificates \
      curl \
      lsb-release \
 && curl -fsSLo /usr/share/keyrings/docker-archive-keyring.asc \
      https://download.docker.com/linux/debian/gpg \
 && echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.asc] https://download.docker.com/linux/debian $(lsb_release -cs) stable" \
      > /etc/apt/sources.list.d/docker.list \
 && apt-get update \
 && apt-get install -y --no-install-recommends docker-ce-cli \
 && rm -rf /var/lib/apt/lists/*

USER jenkins
