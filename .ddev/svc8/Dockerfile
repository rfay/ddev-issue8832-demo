FROM alpine:3.18 AS unused
RUN echo "never built" > /marker.txt

FROM busybox:1.36 AS final
COPY --from=hello-world:latest /hello /hello
RUN echo "I am svc8 (build target, COPY --from an image)" > /marker.txt
CMD ["sleep", "infinity"]

FROM debian:12 AS after-target
