# syntax=docker/dockerfile:1.7
#
# Self-contained, build-from-source image for headscale-admin, used by
# `docker compose build` (so this example needs no prior `make build-docker`).
#
# The build context is the REPO ROOT (see build.context in docker-compose.yaml);
# a repo-root .dockerignore keeps that context lean. This differs from the
# repo-root Dockerfile, which packages an ALREADY-built binary (from
# `make build-docker` / goreleaser); this one builds the frontend and the binary
# end to end.

# --- 1) Frontend: prerender the Svelte SPA into frontend/dist --------------
# The Go build embeds it via //go:embed all:frontend/dist, so it must exist
# before stage 2 compiles.
FROM oven/bun:1 AS frontend
WORKDIR /src/frontend
COPY frontend/package.json frontend/bun.lock ./
RUN bun install --frozen-lockfile
COPY frontend/ ./
RUN bun run build

# --- 2) Backend: static Go binary with the embedded frontend ---------------
# Dependencies are vendored, so the Go build runs offline and reproducibly.
FROM golang:1.26 AS build
WORKDIR /src
COPY . .
COPY --from=frontend /src/frontend/dist ./frontend/dist
ARG VERSION=docker
RUN CGO_ENABLED=0 GOOS=linux go build -mod=vendor \
      -ldflags "-X github.com/YouSysAdmin/headscale-admin/pkg/version.Version=${VERSION}" \
      -o /out/headscale-admin ./cmd/headscale-admin

# --- 3) Runtime: distroless nonroot; /data pre-seeded with the right owner --
FROM busybox:stable AS prep
RUN mkdir -p /out/data/headscale && chown -R 65532:65532 /out/data

FROM gcr.io/distroless/static-debian12:nonroot
WORKDIR /app
COPY --from=build --chown=65532:65532 /out/headscale-admin /app/headscale-admin
COPY --from=prep  --chown=65532:65532 /out/data /data

VOLUME ["/data"]
ENV TZ=UTC
EXPOSE 3000
USER nonroot:nonroot
ENTRYPOINT ["/app/headscale-admin"]
CMD ["serve", "--config", "/config.yaml"]
