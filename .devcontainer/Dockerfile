# Image contents cf. https://github.com/microsoft/vscode-dev-containers/blob/main/containers/go/.devcontainer/base.Dockerfile
ARG VARIANT=1.20-bookworm
FROM --platform=linux/amd64 mcr.microsoft.com/vscode/devcontainers/go:1-${VARIANT}

# Install hugo
RUN wget -O hugo.deb https://github.com/gohugoio/hugo/releases/download/v0.114.1/hugo_0.114.1_linux-amd64.deb
RUN dpkg -i hugo.deb