FROM python:3.12-slim-bookworm@sha256:782412e85d0f0984994c290652577d4018aff08145c85b262bb63dc0c7522254

ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1

WORKDIR /workspace

COPY pyproject.toml README.md ./
COPY src ./src
RUN pip install --no-cache-dir ".[retrieval,ml]"

# cryptography 49's aarch64 wheel uses CPU instructions that are unavailable
# under some Apple Silicon Docker VMs. Keep this in a separate layer so the
# expensive ML dependencies remain cached if the compatibility pin changes.
RUN pip install --no-cache-dir --force-reinstall "cryptography==46.0.3"

EXPOSE 8000
CMD ["lab28", "serve", "--host", "0.0.0.0", "--port", "8000"]
