---
title: 'Inception style builds with private GitHub dependencies'
date: 2024-01-06
permalink: 2024-01-06-inception-style-build/
tags:
- rust
---

# Or "The build within a build within ..."

Recently my colleagues at [SCND](https://www.scnd.com/) and myself found ourselves facing an interesting challenge: building a Docker image of an application written in Rust and depending on Rust libraries in private GitHub repositories; of course not locally, but via GitHub Actions.

So this can be broken down into a couple of smaller challenges:
1. Build a Docker image of a Rust application in GitHub Actions.
1. Depend on a library in a private GitHub repository in GitHub Actions.
1. See how to integrate the two above.

## Docker build

There are fantastic publicly available GitHub Actions to build (and publish) a Docker image, e.g. `docker/metadata-action`, `docker/login-action` and in particular `build-push-action`. Given a `Dockerfile` and the necessary GitHub permissions, a partial workflow could look like this:

```yaml
name: release

on:
  push:
    tags:
      - v*

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - name: Docker metadata
        uses: docker/metadata-action@v5
        id: meta
        with:
          images: ghcr.io/<ORGANIZATION>/${{ github.event.repository.name }}
          tags: type=semver,pattern={{version}}

      - name: Login to GitHub Container Registry
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Docker build and push
        uses: docker/build-push-action@v5
        with:
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          push: true
```

And a `Dockerfile` could look like this:

```dockerfile
ARG RUST_VERSION=1.75.0

FROM rust:${RUST_VERSION}-bookworm AS builder
WORKDIR /app
COPY . .
RUN \
  --mount=type=cache,target=/app/target/ \
  --mount=type=cache,target=/usr/local/cargo/registry/ \
  cargo build --release && \
  cp ./target/release/<APP> /

FROM debian:bookworm-slim AS final
RUN adduser \
  --disabled-password \
  --gecos "" \
  --home "/nonexistent" \
  --shell "/sbin/nologin" \
  --no-create-home \
  --uid "10001" \
  appuser
COPY --from=builder /<APP> /usr/local/bin
RUN chown appuser /usr/local/bin/<APP>
COPY --from=builder /app/config /opt/<APP>/config
RUN chown -R appuser /opt/<APP>
USER appuser
ENV RUST_LOG="<APP>=debug,info"
WORKDIR /opt/<APP>
ENTRYPOINT ["<APP>"]
EXPOSE 8080/tcp
```

Unless the application depends on libraries in private GitHub repositories, the `release` job will execute successfully.

## Dependencies on libraries in private GitHub repository

Locally we can easily depend on libraries in private GitHub repositories using Cargo's `git` dependency feature with ssh URLs:

```
foo = { git = "ssh://git@github.com/<<ORGANIZATION>>/foo" }
```

For this to work, we just need to have our personal public SSH key registered at GitHub, what most every developer has.

If we want CI to be able to build and test applications with such ssh dependencies, we have to put in some more effort. First we need to create a pair of SSH keys. Then we add the public one as deploy key to the GitHub repository which hosts the library dependency. Next we add the private key as action secret to the GitHub repository of the application. And finally we add the fantastic public `webfactory/ssh-agent` GitHub Action to our build:

```yaml
      - name: Install SSH agent
        uses: webfactory/ssh-agent@v0.8.0
        with:
          ssh-private-key: ${{ secrets.<PRIVATE_KEY> }}
```

With all that in place, we can write workflows with jobs/steps to compile, test, etc. our application. But all of that takes place within the GitHub runner. It is still not possible to execute the above `release` job.

Why? Because the `webfactory/ssh-agent` GitHub Action does the "SSH magic", but only within the context of the GitHub runner. When GitHub Actions starts the Docker build which then starts the build of the Rust application – notice the build within a build within ... – neither the SSH public key of the GitHub server hosting the library nor the private key for accessing that repository are available.

## Integration

Luckily it is quite easy to solve the resulting challenge. There are numerous somewhat relevant examples in the internet, but none could be found for our exact use case. Yet we were able to put the pieces together.

First the SSH agent socket needs to be passed down to the Docker build, which can be achieved via the ssh option of the `docker/build-push-action` GitHub action:

```yaml
      - name: Docker build and push
        uses: docker/build-push-action@v5
        with:
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          push: true
          ssh: default=${{ env.SSH_AUTH_SOCK }}
```

This essentially makes the private SSH key for the library in the private GitHub repository available within the Docker build. In order to also make it available one ~~dream~~ build level deeper, i.e. in the Rust build, the respective `RUN` needs to be given the `--mount=type=ssh` option:

```dockerfile
RUN \
  --mount=type=cache,target=/app/target/ \
  --mount=type=cache,target=/usr/local/cargo/registry/ \
  --mount=type=ssh \
  cargo build --release && \
  cp ./target/release/<APP> /
```

Now we are almost there. The last missing piece is to add the public key of the GitHub server to the `known_hosts` file in the builder layer in the Dockerfile, of course before the above build:

```dockerfile
RUN \
  apt-get update && \
  apt-get install -y jq && \
  mkdir -p -m 0700 ~/.ssh && \
  curl --silent https://api.github.com/meta  | jq --raw-output '"github.com "+.ssh_keys[]' >> ~/.ssh/known_hosts && \
  chmod 600 ~/.ssh/known_hosts
```

Phew, we are finally there! So it turns out that the essence of making this work is to pass the private SSH key down from the "top level" build, where it is defined as an action secret, to the Docker build from where it needs to be passed down to the Rust build. Inception at its best! Of course we also must not forget to add the public key of the GitHub server to `known_hosts`.
