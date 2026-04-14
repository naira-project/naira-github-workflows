# SPDX-FileCopyrightText: 2026 Naira Project Contributors
# SPDX-License-Identifier: Apache-2.0
#
# Multi-stage, multi-platform Dockerfile for Go services.
# Copy to your service repo and adjust MODULE and BINARY as needed.

# ── Stage 1: Build ────────────────────────────────────────────────────────
FROM --platform=$BUILDPLATFORM golang:1.22-alpine AS builder

# Build arguments injected by the container workflow
ARG TARGETOS=linux
ARG TARGETARCH=amd64
ARG VERSION=dev
ARG COMMIT=unknown
ARG BUILD_DATE

WORKDIR /app

# Download dependencies first (layer-cached separately from source)
COPY go.mod go.sum ./
RUN go mod download && go mod verify

# Copy source
COPY . .

# Build the binary
RUN CGO_ENABLED=0 GOOS=$TARGETOS GOARCH=$TARGETARCH \
    go build \
      -trimpath \
      -ldflags="-s -w \
        -X main.version=${VERSION} \
        -X main.commit=${COMMIT} \
        -X main.buildDate=${BUILD_DATE}" \
      -o /app/server \
      ./cmd/server   # ← adjust to your main package path

# ── Stage 2: Security scan (optional, uncomment to enable) ───────────────
# FROM aquasec/trivy:latest AS scanner
# COPY --from=builder /app/server /app/server
# RUN trivy fs --exit-code 1 --severity HIGH,CRITICAL /app/server

# ── Stage 3: Runtime ──────────────────────────────────────────────────────
# distroless/static: no shell, no package manager — minimal attack surface
FROM gcr.io/distroless/static:nonroot AS runtime

# OCI standard labels (populated by docker/metadata-action in CI)
LABEL org.opencontainers.image.source="https://github.com/naira-project/REPO_NAME"
LABEL org.opencontainers.image.licenses="Apache-2.0"
LABEL org.opencontainers.image.vendor="Naira Project"

# Copy the binary from builder
COPY --from=builder /app/server /server

# Run as non-root user (uid 65532 = nonroot in distroless)
USER nonroot:nonroot

EXPOSE 8080

ENTRYPOINT ["/server"]
