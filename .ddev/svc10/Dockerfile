FROM base
COPY --from=files local.txt /local.txt
RUN echo "I am svc10 (additional_contexts)" > /marker.txt
CMD ["sleep", "infinity"]
