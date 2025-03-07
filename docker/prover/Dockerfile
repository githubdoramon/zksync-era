# Will work locally only after prior contracts build and universal setup key download

FROM nvidia/cuda:11.8.0-devel-ubuntu22.04 as builder

ARG DEBIAN_FRONTEND=noninteractive

ARG ERA_BELLMAN_CUDA_RELEASE=latest

RUN apt-get update && apt-get install -y curl jq clang openssl libssl-dev gcc g++ \
    pkg-config build-essential libclang-dev && \
    rm -rf /var/lib/apt/lists/*

ENV RUSTUP_HOME=/usr/local/rustup \
    CARGO_HOME=/usr/local/cargo \
    PATH=/usr/local/cargo/bin:$PATH

RUN curl https://sh.rustup.rs -sSf | bash -s -- -y && \
    rustup install nightly-2023-07-21 && \
    rustup default nightly-2023-07-21

WORKDIR /usr/src/zksync

ENV GITHUB_OWNER=matter-labs
ENV GITHUB_REPO=era-bellman-cuda

RUN set -e; \
    if [ "$ERA_BELLMAN_CUDA_RELEASE" = "latest" ]; then \
    latest_release_data=$(curl --silent "https://api.github.com/repos/${GITHUB_OWNER}/${GITHUB_REPO}/releases"); \
    latest_tag=$(echo "$latest_release_data" | jq -r '.[0].tag_name'); \
    source_url="https://github.com/${GITHUB_OWNER}/${GITHUB_REPO}/archive/refs/tags/${latest_tag}.tar.gz"; \
    binary_url="https://github.com/${GITHUB_OWNER}/${GITHUB_REPO}/releases/download/${latest_tag}/bellman-cuda.tar.gz"; \
    else \
    source_url="https://github.com/${GITHUB_OWNER}/${GITHUB_REPO}/archive/refs/tags/${ERA_BELLMAN_CUDA_RELEASE}.tar.gz"; \
    binary_url="https://github.com/${GITHUB_OWNER}/${GITHUB_REPO}/releases/download/${ERA_BELLMAN_CUDA_RELEASE}/bellman-cuda.tar.gz"; \
    fi; \
    curl --silent --location "$source_url" --output bellman-cuda-source.tar.gz; \
    curl --silent --location "$binary_url" --output bellman-cuda.tar.gz; \
    mkdir -p bellman-cuda; \
    tar xvfz bellman-cuda.tar.gz -C ./bellman-cuda; \
    tar xvfz bellman-cuda-source.tar.gz -C ./bellman-cuda --strip-components=1

ENV BELLMAN_CUDA_DIR=/usr/src/zksync/bellman-cuda

COPY . .

RUN cargo build --release

FROM nvidia/cuda:11.8.0-runtime-ubuntu22.04 as runner

ARG DEBIAN_FRONTEND=noninteractive

RUN apt-get update && apt-get install -y libpq5 ca-certificates openssl && rm -rf /var/lib/apt/lists/*

COPY etc/system-contracts/bootloader/build/artifacts/ /etc/system-contracts/bootloader/build/artifacts/
COPY etc/system-contracts/artifacts-zk /etc/system-contracts/artifacts-zk
COPY contracts/ethereum/artifacts/ /contracts/ethereum/artifacts/
COPY contracts/zksync/artifacts-zk/ /contracts/zksync/artifacts-zk/
COPY setup_2\^26.key /etc/

COPY core/bin/verification_key_generator_and_server/data/ /core/bin/verification_key_generator_and_server/data/

COPY --from=builder /usr/src/zksync/target/release/zksync_prover /usr/bin/

ENTRYPOINT ["zksync_prover"]
