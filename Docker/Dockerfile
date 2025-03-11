# Dockerfile for ceph-nano setup
FROM ubuntu

# Install ca-certificates, curl, and Docker
RUN apt-get update && \
    apt-get install -y ca-certificates curl && \
    curl -fsSL https://get.docker.com | sed 's/sleep [0-9]+/sleep .1/g' | sh

# Download and set up Ceph Nano CLI
RUN echo 'Downloading Ceph Nano CLI...' && \
    curl -L https://github.com/ceph/cn/releases/download/v2.3.1/cn-v2.3.1-linux-amd64 -o /usr/local/sbin/cn && \
    chmod +x /usr/local/sbin/cn