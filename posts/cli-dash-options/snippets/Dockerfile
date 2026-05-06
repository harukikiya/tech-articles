FROM debian:bookworm-slim
RUN apt-get update && apt-get install -y \
    gcc \
    python3 \
    grep \
    coreutils \
 && rm -rf /var/lib/apt/lists/*
WORKDIR /work