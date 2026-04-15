# syntax=docker/dockerfile:1.6

ARG GO_VERSION=1.22
ARG ALPINE_VERSION=3.20

FROM golang:${GO_VERSION}-alpine${ALPINE_VERSION} AS builder

ENV CGO_ENABLED=0 \
    GOOS=linux \
    GOFLAGS=-mod=readonly

RUN apk add --no-cache ca-certificates tzdata git

WORKDIR /src

COPY go.mod go.sum ./
RUN --mount=type=cache,target=/go/pkg/mod \
    go mod download

COPY . .

RUN --mount=type=cache,target=/go/pkg/mod \
    --mount=type=cache,target=/root/.cache/go-build \
    go build -trimpath -ldflags="-s -w" -o /out/api ./cmd/api

FROM alpine:${ALPINE_VERSION} AS runtime

RUN apk add --no-cache --upgrade ca-certificates tzdata wget && \
    addgroup -S app -g 10001 && \
    adduser  -S app -u 10001 -G app -h /app -s /sbin/nologin

WORKDIR /app

COPY --from=builder --chown=app:app /out/api /app/api

USER app:app

EXPOSE 8081

HEALTHCHECK --interval=15s --timeout=3s --start-period=5s --retries=3 \
    CMD wget -qO- http://127.0.0.1:8081/health || exit 1

ENTRYPOINT ["/app/api"]
