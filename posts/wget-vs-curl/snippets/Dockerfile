FROM debian:bookworm-slim
RUN apt-get update && apt-get install -y \
    curl \
    wget \
    ca-certificates \
    jq \
 && rm -rf /var/lib/apt/lists/*
WORKDIR /work