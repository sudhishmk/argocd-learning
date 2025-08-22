# Start from the Red Hat Universal Base Image
FROM registry.access.redhat.com/ubi9/ubi:9.6-1755678605

# Switch to the root user temporarily to install tools
USER root

# Install wget and tar, then download and install Helm v2
RUN dnf install -y wget tar gzip && \
    wget -qO- https://get.helm.sh/helm-v2.12.3-linux-amd64.tar.gz | tar -xvzf - && \
    mv linux-amd64/helm /usr/local/bin/helm && \
    dnf clean all

# IMPORTANT: Switch back to a non-root user to run securely
USER 1001
