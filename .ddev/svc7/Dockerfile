FROM alpine:3.20 AS base-amd64
FROM alpine:3.19 AS base-arm64
FROM base-${TARGETARCH}
RUN echo "I am svc7 (stage picked by TARGETARCH)" > /marker.txt
CMD ["sleep", "infinity"]
