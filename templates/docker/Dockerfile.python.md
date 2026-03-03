# Шаблон Dockerfile для Python приложения
# Использование:
#   docker build -t myapp .
#   docker run -p 8000:8000 myapp

FROM python:3.11-slim AS builder

# Устанавливаем системные зависимости (если нужны)
RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /app

# Копируем зависимости
COPY requirements.txt .

# Создаем виртуальное окружение и устанавливаем зависимости
RUN python -m venv /venv && \
    /venv/bin/pip install --no-cache-dir -r requirements.txt

# Финальный образ
FROM python:3.11-slim

# Копируем виртуальное окружение из builder
COPY --from=builder /venv /venv

# Добавляем venv в PATH
ENV PATH="/venv/bin:$PATH"

# Создаем непривилегированного пользователя
RUN addgroup --system --gid 1001 appgroup && \
    adduser --system --uid 1001 --gid 1001 appuser

WORKDIR /app

# Копируем код приложения
COPY --chown=appuser:appgroup . .

# Переключаемся на непривилегированного пользователя
USER appuser

# Порт приложения
EXPOSE 8000

# Healthcheck
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD python -c "import urllib.request; urllib.request.urlopen('http://localhost:8000/health')"

# Запуск
CMD ["gunicorn", "--bind", "0.0.0.0:8000", "app:app"]