FROM mcr.microsoft.com/devcontainers/base:ubuntu

# Target architecture injected by buildx (e.g. amd64, arm64)
ARG TARGETARCH

# Avoid prompts from apt
ENV DEBIAN_FRONTEND=noninteractive

# Install base packages
RUN apt-get update && apt-get install -y \
    curl \
    wget \
    git \
    vim \
    bash-completion \
    jq \
    make \
    build-essential \
    ca-certificates \
    gnupg \
    lsb-release \
    erofs-utils \
    fuse-overlayfs \
    && rm -rf /var/lib/apt/lists/*

# Install Node.js (required for npm-based tool installs below)
RUN curl -fsSL https://deb.nodesource.com/setup_lts.x | bash - \
    && apt-get install -y nodejs \
    && rm -rf /var/lib/apt/lists/*

# Install kind
RUN curl -Lo /usr/local/bin/kind "https://kind.sigs.k8s.io/dl/latest/kind-linux-${TARGETARCH}" && \
    chmod +x /usr/local/bin/kind

# Install k9s
RUN curl -sL "https://github.com/derailed/k9s/releases/latest/download/k9s_Linux_${TARGETARCH}.tar.gz" | \
    tar xz -C /usr/local/bin k9s && \
    chmod +x /usr/local/bin/k9s

# Install Flux CLI
RUN curl -s https://fluxcd.io/install.sh | bash

# Install Marp CLI
RUN npm install -g @marp-team/marp-cli

# Install markdown tools
RUN npm install -g \
    markdownlint-cli \
    markdown-link-check \
    prettier

# Install YAML tools (kubeconform replaces the archived kubeval)
RUN wget -qO- "https://github.com/yannh/kubeconform/releases/latest/download/kubeconform-linux-${TARGETARCH}.tar.gz" | \
    tar xz -C /usr/local/bin kubeconform && \
    chmod +x /usr/local/bin/kubeconform

# Install yq for YAML processing
RUN wget "https://github.com/mikefarah/yq/releases/latest/download/yq_linux_${TARGETARCH}" -O /usr/local/bin/yq && \
    chmod +x /usr/local/bin/yq

# Set up bash completion for tools available at build time
RUN echo 'source /etc/bash_completion' >> /etc/bash.bashrc && \
    kind completion bash > /etc/bash_completion.d/kind && \
    flux completion bash > /etc/bash_completion.d/flux
# Note: kubectl and helm completions are added in post-create.sh (installed via features)

# Create helpful aliases for vscode user
RUN echo 'alias k=kubectl' >> /home/vscode/.bashrc && \
    echo 'alias kgp="kubectl get pods"' >> /home/vscode/.bashrc && \
    echo 'alias kgs="kubectl get svc"' >> /home/vscode/.bashrc && \
    echo 'alias kgd="kubectl get deploy"' >> /home/vscode/.bashrc && \
    echo 'alias kga="kubectl get all"' >> /home/vscode/.bashrc && \
    echo 'alias kdesc="kubectl describe"' >> /home/vscode/.bashrc && \
    echo 'alias klogs="kubectl logs"' >> /home/vscode/.bashrc

# Set working directory
WORKDIR /workspaces

# Clean up
RUN apt-get clean && rm -rf /var/lib/apt/lists/*

ENV DEBIAN_FRONTEND=dialog
