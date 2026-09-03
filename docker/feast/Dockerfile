FROM python:3.12.11-slim

ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1

RUN pip install --no-cache-dir "feast==0.66.0" "redis==6.4.0"

# Feast's feature server is FastAPI/Starlette under the hood. Auto-instrument
# it so its own request handling shows up as a distinct OTEL service on the
# trace IT-trace-span-coverage checks, instead of leaving Feast unobserved.
# gunicorn forks a worker after boot, and a gRPC channel opened before fork()
# does not survive the fork — spans queue but silently never export. The HTTP
# exporter opens its connection lazily per export, so it works after fork.
RUN pip install --no-cache-dir "opentelemetry-distro" "opentelemetry-exporter-otlp-proto-http" \
    && opentelemetry-bootstrap -a install

WORKDIR /workspace/feature-repo

EXPOSE 6566 8000
