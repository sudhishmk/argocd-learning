# Start from the Red Hat Universal Base Image
FROM registry.access.redhat.com/ubi9/ubi:9.6-1755678605

# Switch to the root user temporarily to install tools
USER root

# Install wget and tar, then download and install Helm v2
RUN dnf install -y wget tar gzip && \
    mkdir /custom-tools && \
    wget -qO /custom-tools/argocd-vault-plugin https://github.com/argoproj-labs/argocd-vault-plugin/releases/download/v1.6.0/argocd-vault-plugin_1.6.0_linux_amd64 && \
    chmod +x /custom-tools/argocd-vault-plugin && \
    dnf clean all

# IMPORTANT: Switch back to a non-root user to run securely
USER 1001
