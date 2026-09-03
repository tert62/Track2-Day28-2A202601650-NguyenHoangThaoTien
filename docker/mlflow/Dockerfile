FROM ghcr.io/mlflow/mlflow:v3.15.2

# cryptography 49's aarch64 wheel raises SIGILL under some Apple Silicon
# Docker VMs. MLflow imports Fernet when its server CLI loads the SQL store.
RUN pip install --no-cache-dir --force-reinstall "cryptography==46.0.3"
