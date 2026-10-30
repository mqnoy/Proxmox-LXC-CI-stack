FROM --platform=linux/amd64 docker.io/library/golang:1.23-alpine AS builder

ARG TARGETOS=linux
ARG TARGETARCH=arm64

WORKDIR /src

COPY go.mod ./
COPY cmd/ ./cmd/

RUN CGO_ENABLED=0 \
    GOOS=${TARGETOS} \
    GOARCH=${TARGETARCH} \
    go build -trimpath -ldflags="-s -w" \
    -o /out/api ./cmd/api

FROM scratch

COPY --from=builder /out/api /api

USER 10001:10001

EXPOSE 8080

ENTRYPOINT ["/api"]