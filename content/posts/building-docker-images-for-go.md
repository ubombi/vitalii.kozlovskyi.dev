---
title: "Building multiarch docker images for Go"
date: 2022-04-05T08:03:26+02:00
draft: false 
summary: "How to build small multiarch image using Github Workflow"
---

### 1. Order matters
Docker rebuilds images layer by layer, starting with first change. So

```docker
FROM golang:1.17

# modules cached in a separate layer
# it would redownload modules, only if dependencies were changed
WORKDIR /go/src/foo/
COPY go.mod go.sum ./
RUN go mod download

# copy verything else & build static binary
COPY . .
RUN CGO_ENABLED=0 go build -o /bin/foo -a ./

ENTRYPOINT ["/bin/foo"]
```

### 2. Split stages
One can run build, code-generation, chore tasks in a `prep`, `build` stages, while keeping result clean.
Resulting image will be few megabytes in size.

```dockerfile
FROM golang:1.17-alpine AS build

# Download fresh TLS certificates
RUN apk add --no-cache ca-certificates 
RUN update-ca-certificates

# modules cached in a separate layer
WORKDIR /go/src/foo/
COPY go.mod go.sum ./
RUN go mod download

# build static binary
COPY . .
RUN CGO_ENABLED=0 go build -o /bin/foo -a ./

# This results in a single layer image which consists from binary and CA TLS certificates
FROM scratch
COPY --from=build /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/
COPY --from=build /bin/foo /bin/foo

ENTRYPOINT ["/bin/foo"]
```

### 3. Be aware of build platform and target platform.
`docker buildx ...` multi-arch build can be done either with crosscompiling or emulation.
Go can build binaries for almost any platform, so we would stick with crosscompiling, since it's way faster.
`Dockerfile`
```docker
ARG GOVERSION=1.17
ARG BUILDPLATFORM="linux/amd64"

# specify that regardless of target, we use image native to build platform.
FROM --platform=$BUILDPLATFORM golang:${GOVERSION}-alpine AS build
RUN apk add --no-cache ca-certificates 
RUN update-ca-certificates

# modules cached in a separate layer
WORKDIR /go/src/foo/
COPY go.mod go.sum ./
RUN go mod download

# build static binary
COPY . .

# after this line, layers would be splited for each target platform.
ARG TARGETOS TARGETARCH

# specify default target ${FOO:-default value}, so regular docker builds will remain working. 
RUN CGO_ENABLED=0 GOOS=${TARGETOS:-linux} GOARCH=${TARGETARCH:-amd64} \
    go build -o /bin/foo -a ./

# This results in a single layer image
FROM scratch

# We do not use following args, but It's important to accept same them here.
# Otherwise layer cache is broken, and all architectures will get same binary
ARG TARGETOS TARGETARCH

COPY --from=build /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/
COPY --from=build /bin/foo /bin/foo
ENTRYPOINT ["/bin/foo"]

```

### 4. Enable Github Workflow to build & push your containers
Specify secrets of your container registry (E.G: Dockerhub, ECR, or GHCR)

`.github/workflows/push.yaml`
```yaml
name: build and push latest image

on:
  push:
    branches:
      - 'master'

jobs:
  docker:
    runs-on: ubuntu-latest
    steps:
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v1
      - name: Login to DockerHub
        uses: docker/login-action@v1 
        with:
          username: ${{ secrets.DOCKERHUB_USER }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}
      - name: Build and push
        uses: docker/build-push-action@v2
        with:
          push: true
          platforms: linux/amd64,linux/arm64
          tags: foo/bar:latest
          build-args: |
            COMMIT=${{ github.sha }}
```

Here you can pass additional information, via `build-args`.
