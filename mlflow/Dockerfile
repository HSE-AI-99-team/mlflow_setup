FROM python:3.11-slim

# Системные зависимости (psycopg2 требует libpq)
RUN apt-get update && apt-get install -y --no-install-recommends \
        libpq-dev \
        gcc \
    && rm -rf /var/lib/apt/lists/*

# Устанавливаем MLflow + драйвер PostgreSQL + boto3 (для S3/MinIO)
RUN pip install --no-cache-dir \
        mlflow==3.12.0 \
        psycopg2-binary \
        boto3

EXPOSE 5000

# Команда переопределяется в docker-compose.yml
CMD ["mlflow", "server", "--host", "0.0.0.0", "--port", "5000"]
