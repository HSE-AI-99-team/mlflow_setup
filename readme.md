# MLflow Tracking Infrastructure
+
+## Архитектура
+
+```
+Локальный ноутбук (mlflow SDK)
+        │
+        │  HTTP :5000  (MLflow Tracking API)
+        ▼
+┌─────────────────────┐
+│  MLflow Server      │  — логирует метрики/параметры
+│  (mlflow:2.13.2)    │  — проксирует артефакты
+└──────┬──────────────┘
+       │                      │
+       ▼                      ▼
+┌─────────────┐      ┌──────────────────┐
+│ PostgreSQL  │      │  MinIO (S3)      │
+│ (metadata)  │      │  (artifacts)     │
+└─────────────┘      └──────────────────┘
+```
+
+> Все сервисы работают в одной Docker-сети `mlflow_net` на VPS.
+> Наружу открыты только три порта: **5000** (MLflow UI/API), **9000** (MinIO S3 API), **9001** (MinIO Web Console).
+
+---
+
+## Структура проекта
+
+```
+.
+├── docker-compose.yml   # главный файл оркестрации
+├── env.example          # шаблон переменных окружения
+├── mlflow/
+│   └── Dockerfile       # образ MLflow-сервера
+└── readme.md
+```
+
+---
+
+## Деплой на VPS (один раз)
+
+### 1. Клонировать репозиторий
+```bash
+git clone <repo_url> mlflow_setup && cd mlflow_setup
+```
+
+### 2. Создать `.env` из шаблона и задать пароли
+```bash
+cp env.example .env
+nano .env          # отредактируйте все значения *change_me*
+```
+
+> ⚠️ Никогда не коммитьте `.env` в git — он уже добавлен в `.gitignore`.
+
+### 3. Запустить инфраструктуру
+```bash
+docker compose up -d --build
+```
+
+### 4. Проверить статус сервисов
+```bash
+docker compose ps
+docker compose logs -f mlflow
+```
+
+После успешного старта:
+| Сервис | URL |
+|---|---|
+| MLflow UI | `http://<VPS_IP>:5000` |
+| MinIO Console | `http://<VPS_IP>:9001` |
+
+---
+
+## Подключение с локального ноутбука
+
+### Установить зависимости
+```bash
+pip install mlflow boto3
+```
+
+### Пример трекинга эксперимента
+```python
+import os
+import mlflow
+
+# ── Credentials ──────────────────────────────────────────────
+os.environ["MLFLOW_TRACKING_URI"]   = "http://<VPS_IP>:5000"
+os.environ["MLFLOW_S3_ENDPOINT_URL"] = "http://<VPS_IP>:9000"
+os.environ["AWS_ACCESS_KEY_ID"]      = "<MINIO_ROOT_USER>"      # из .env
+os.environ["AWS_SECRET_ACCESS_KEY"]  = "<MINIO_ROOT_PASSWORD>"  # из .env
+
+# ── Эксперимент ───────────────────────────────────────────────
+mlflow.set_experiment("my-experiment")
+
+with mlflow.start_run():
+    mlflow.log_param("learning_rate", 0.01)
+    mlflow.log_metric("accuracy", 0.95)
+    mlflow.log_artifact("model.pkl")   # файл загрузится в MinIO
+```
+
+> Credentials удобно вынести в `~/.bashrc` / `.zshrc` или в файл `.env` на ноутбуке
+> и подгружать через `python-dotenv`.
+
+---
+
+## Управление инфраструктурой
+
+| Действие | Команда |
+|---|---|
+| Запустить | `docker compose up -d` |
+| Остановить (данные сохраняются) | `docker compose stop` |
+| Пересобрать образ MLflow | `docker compose up -d --build mlflow` |
+| Посмотреть логи | `docker compose logs -f` |
+| Полное удаление с данными | `docker compose down -v` |
+
+---
+
+## Описание переменных окружения (`.env`)
+
+| Переменная | Описание |
+|---|---|
+| `POSTGRES_USER` | Имя пользователя БД |
+| `POSTGRES_PASSWORD` | Пароль пользователя БД |
+| `POSTGRES_DB` | Название базы данных |
+| `POSTGRES_PORT` | Порт PostgreSQL (внутренний, `5432`) |
+| `MINIO_ROOT_USER` | Логин администратора MinIO |
+| `MINIO_ROOT_PASSWORD` | Пароль администратора MinIO (мин. 8 символов) |
+| `MINIO_PORT` | S3 API порт (`9000`) |
+| `MINIO_CONSOLE_PORT` | Порт веб-консоли MinIO (`9001`) |
+| `MINIO_BUCKET_NAME` | Имя бакета для артефактов MLflow |
+| `MLFLOW_PORT` | Порт MLflow Tracking Server (`5000`) |
+| `MLFLOW_S3_ENDPOINT_URL` | URL MinIO для MLflow-сервера (внутри сети: `http://minio:9000`) |
+| `AWS_ACCESS_KEY_ID` | = `MINIO_ROOT_USER` |