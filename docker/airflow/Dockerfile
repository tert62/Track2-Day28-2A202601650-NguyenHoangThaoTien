FROM apache/airflow:3.3.1-python3.12

COPY --chown=airflow:root docker/airflow/requirements.txt /tmp/lab28-requirements.txt
RUN pip install --no-cache-dir -r /tmp/lab28-requirements.txt

# See docker/api/Dockerfile: keep the pin in a separate layer so changing it
# does not invalidate the expensive Airflow/PySpark dependency layer.
RUN pip install --no-cache-dir --force-reinstall "cryptography==46.0.3"

COPY --chown=airflow:root pyproject.toml README.md /opt/lab28/
COPY --chown=airflow:root src /opt/lab28/src
RUN pip install --no-cache-dir --no-deps /opt/lab28
