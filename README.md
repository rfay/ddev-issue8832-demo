# DDEV #8832 demo: custom build-service base image collisions

This project reproduces [ddev/ddev#8832](https://github.com/ddev/ddev/issues/8832):
when more than one custom compose service in a DDEV project uses `build:`
against the same default base image, DDEV's documented naming convention
(`image: ${BASE_IMAGE:-example:latest}-${DDEV_SITENAME}-built`) makes the
built images collide on one Docker tag, so the second build silently
overwrites the first — containers end up running the wrong image.

The first three services all build from `ubuntu:24.04` in the three ways DDEV
supports `build:`, each with a per-service tag qualifier that avoids the
collision (`-svc1-`, `-svc2-`, `-svc3-`). `svc4` and `svc5` cover Dockerfile
base-image resolution edge cases, and `svc6` through `svc14` are described in
[section 5](#5-confirm-buildkits-resolution-rules):

| Service | Build style | File |
| --- | --- | --- |
| `svc1` | separate `Dockerfile` | [.ddev/svc1/Dockerfile](.ddev/svc1/Dockerfile) |
| `svc2` | separate `Dockerfile` | [.ddev/svc2/Dockerfile](.ddev/svc2/Dockerfile) |
| `svc3` | `dockerfile_inline` (the pattern from the [custom-compose-files docs](https://ddev.readthedocs.io/en/stable/users/extend/custom-compose-files/)) | [.ddev/docker-compose.svc3.yaml](.ddev/docker-compose.svc3.yaml) |
| `svc4` | `build.args` value supplied from the project environment | [.ddev/docker-compose.svc4.yaml](.ddev/docker-compose.svc4.yaml) |
| `svc5` | multi-stage Dockerfile with a later stage named after an earlier base image | [.ddev/svc5/Dockerfile](.ddev/svc5/Dockerfile) |

Each container writes a distinct `/marker.txt` at build time, so you can tell
whether each service is running its own image or has been overwritten by
another service's build.

Before running sections 1 through 3, set the build argument required by
`svc4`:

```bash
export BASE_IMAGE=ubuntu:24.04
```

## 1. Reproduce the tag collision (any DDEV version)

The collision itself is a plain Docker fact, not something a DDEV version
changes: if two services' `image:` tags are literally identical, only one
entry can ever exist in `docker images`, and whichever service built last
"wins" for both containers.

To see it, edit all three `.ddev/docker-compose.svc*.yaml` files and remove
each service's name from the tag, so all three read exactly the same,
matching DDEV's documented convention literally:

```diff
- image: ${BASE_IMAGE:-ubuntu:24.04}-${DDEV_SITENAME}-svc1-built
+ image: ${BASE_IMAGE:-ubuntu:24.04}-${DDEV_SITENAME}-built
```

(do this for `svc1`, `svc2`, and `svc3` — for `svc3` the variable is
`BASE_IMAGE` too, see the file)

Then:

```bash
ddev restart
docker images | grep issue8832-demo   # only ONE ubuntu-based image, not three
docker exec ddev-issue8832-demo-svc1 cat /marker.txt
docker exec ddev-issue8832-demo-svc2 cat /marker.txt
docker exec ddev-issue8832-demo-svc3 cat /marker.txt
# all three print the same marker — whichever service built last
```

Revert the edit (`git checkout .ddev`) before moving on.

## 2. Reproduce the broken pre-pull/describe behavior (DDEV v1.25.4 and earlier)

With the repo's default, already-unique tags (`-svc1-`, `-svc2-`, `-svc3-`),
the collision above doesn't happen — but on DDEV v1.25.4 and earlier this
"obvious" fix breaks a different thing: DDEV finds the image to pre-pull by
string-trimming the `-${DDEV_SITENAME}-built` suffix off the tag, and doesn't
know how to remove a service-name segment too, so it tries to pull the local
tag itself as if it were a real registry reference.

Install DDEV v1.25.4 (or any released version without the #8832 fix), then:

```bash
ddev start
# non-fatal warning: Unable to pull Docker images: DDEV tries to pull the
# local image tag of every service, svc1 through svc14, including
# "docker.io/library/ubuntu:24.04-issue8832-demo-svc1"

ddev utility download-images
# hard failure, with the same local image references
```

The project still runs (the warning is non-fatal at `start`), but offline
pre-caching is broken for these services.

## 3. Confirm the fix

Build a `ddev` binary from the branch/PR that fixes #8832 (resolves the base
image by reading the Dockerfile/`dockerfile_inline` and `build.args` directly,
instead of parsing it out of the tag), put it first on `PATH`, then:

```bash
ddev start
# no pull warning

ddev describe -j | jq -r '.raw.services | to_entries | map(select(.key|test("^svc[1-3]$"))) | sort_by(.key|ltrimstr("svc")|tonumber)[] | "\(.key): \(.value.image)"'
# svc1: ubuntu:24.04
# svc2: ubuntu:24.04
# svc3: ubuntu:24.04

ddev utility download-images
# success: ubuntu:24.04 is pulled once for svc1 through svc4, and each
# other service pulls the images listed in sections 4 and 5
```

## 4. Confirm environment-supplied build args and stage-name handling

`svc4` has an arbitrary local image tag and supplies `BASE_IMAGE` without a
value. Docker Compose resolves it from the project environment at build time.
`svc5` first uses `alpine:3.20`, then names a later `busybox:1.36` stage
`alpine`. Only previously declared stage names are internal references, and
the `alpine:3.20` stage is never built because the final stage doesn't use it,
so only `busybox:1.36` is pulled.

With the #8832 branch binary first on `PATH`, run:

```bash
BASE_IMAGE=alpine:3.20 ddev start
ddev describe -j | jq -r '.raw.services | to_entries | map(select(.key|test("^svc[45]$"))) | sort_by(.key|ltrimstr("svc")|tonumber)[] | "\(.key): \(.value.image)"'
# svc4: alpine:3.20
# svc5: busybox:1.36

BASE_IMAGE=alpine:3.20 ddev utility download-images
# success: svc4 pulls alpine:3.20 and svc5 pulls busybox:1.36
```

## 5. Confirm BuildKit's resolution rules

`svc6` through `svc14` use Dockerfile and compose features where the base
image can only be found by resolving the Dockerfile the way BuildKit does:

| Service | Case | Pulled |
| --- | --- | --- |
| `svc6` | `build.args` sets `TAG`, which a later `ARG BASE=alpine:${TAG}` uses | `alpine:3.20`, not the `3.19` default |
| `svc7` | `FROM base-${TARGETARCH}` picks a stage by the Docker daemon's architecture | `alpine:3.20` on amd64, `alpine:3.19` on arm64 |
| `svc8` | `target: final`, with `COPY --from=hello-world:latest` | `busybox:1.36`, `hello-world:latest`; not the stages outside the target |
| `svc9` | `FROM scratch` with `COPY --from=busybox:1.36-musl` | `busybox:1.36-musl`; `scratch` is never pulled |
| `svc10` | `FROM base` and `COPY --from=files`, both compose `additional_contexts` | `alpine:3.20` from `docker-image://alpine:3.20`; not `base` or `files` |
| `svc11` | `RUN --mount=from=busybox:1.36-musl` | `alpine:3.20`, `busybox:1.36-musl` |
| `svc12` | absolute `dockerfile:` path outside the build context | `debian:12-slim` |
| `svc13` | `FROM base`, where `base` is `service:svc12` in compose `additional_contexts` | nothing; `ddev describe` shows svc12's local tag |
| `svc14` | service-level `platform: linux/amd64` with `FROM base-${TARGETOS}-${TARGETARCH}` | `alpine:3.20` on every host, not `alpine:3.19` on arm64 or `base-darwin-arm64` on macOS |

On an arm64 host, `svc14` needs amd64 emulation, which Docker Desktop and
OrbStack provide.

```bash
ddev describe -j | jq -r '.raw.services | to_entries | map(select(.key|test("^svc([6-9]|1[0-4])$"))) | sort_by(.key|ltrimstr("svc")|tonumber)[] | "\(.key): \(.value.image)"'
# svc6: alpine:3.20
# svc7: alpine:3.20
# svc8: busybox:1.36, hello-world:latest
# svc9: busybox:1.36-musl
# svc10: alpine:3.20
# svc11: alpine:3.20, busybox:1.36-musl
# svc12: debian:12-slim
# svc13: issue8832-absolute-dockerfile-local-tag:latest
# svc14: alpine:3.20

ddev utility download-images
# succeeds; a resolver that doesn't expand platform ARGs fails here with
# "invalid reference format" on the image 'base-' or 'base--'
```

## Cleanup

```bash
ddev delete -O
```
