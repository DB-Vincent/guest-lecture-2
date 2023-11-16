FROM redhat/ubi9-minimal:latest

RUN microdnf install wget tar gzip -y && \
    cd /tmp/ && \
    wget https://github.com/americanexpress/baton/releases/download/v0.1/linux_amd64.tar.gz && \
    tar -xvf linux_amd64.tar.gz && \
    mv linux_amd64/baton /usr/local/bin/ && \
    chmod +x /usr/local/bin/baton

ENTRYPOINT ["/usr/local/bin/baton"]