FROM alpine:3.20 AS base-linux-amd64
FROM alpine:3.19 AS base-linux-arm64
FROM base-${TARGETOS}-${TARGETARCH}
RUN echo "I am svc14 (stage picked by the service's platform)" > /marker.txt
CMD ["sleep", "infinity"]
