# ---------- БАЗОВЫЙ ОБРАЗ ----------
# Используем официальный образ с конкретной версией (не latest!)
ARG NODE_VERSION=18.17.0
FROM node:${NODE_VERSION}-alpine AS builder

# ---------- МЕТАДАННЫЕ ----------
LABEL maintainer="your-email@example.com"
LABEL version="1.0.0"
LABEL description="Production-ready Node.js application"

# ---------- ПЕРЕМЕННЫЕ СБОРКИ ----------
# ARG доступны только во время сборки
ARG APP_USER=nodeuser
ARG APP_HOME=/app
ARG NODE_ENV=production

# ENV доступны в контейнере постоянно
ENV NODE_ENV=${NODE_ENV} \
    PATH="${APP_HOME}/node_modules/.bin:${PATH}" \
    # Настройки Node.js
    NODE_OPTIONS="--max-old-space-size=512" \
    # Настройки npm
    npm_config_cache=/tmp/.npm

# ---------- СОЗДАНИЕ ПОЛЬЗОВАТЕЛЯ ----------
# Лучше не запускать приложения от root
RUN addgroup -g 1001 -S ${APP_USER} && \
    adduser -u 1001 -S ${APP_USER} -G ${APP_USER}

# ---------- УСТАНОВКА СИСТЕМНЫХ ЗАВИСИМОСТЕЙ ----------
RUN apk add --no-cache \
    curl \
    tzdata \
    && cp /usr/share/zoneinfo/Europe/Moscow /etc/localtime \
    && echo "Europe/Moscow" > /etc/timezone

# ---------- РАБОЧАЯ ДИРЕКТОРИЯ ----------
WORKDIR ${APP_HOME}

# ---------- КОПИРОВАНИЕ ЗАВИСИМОСТЕЙ ----------
# Сначала только package.json для кэширования слоя
COPY package*.json ./
COPY yarn.lock* ./

# ---------- УСТАНОВКА ЗАВИСИМОСТЕЙ ----------
RUN npm ci --only=production --no-audit --no-fund && \
    npm cache clean --force

# ---------- КОПИРОВАНИЕ ИСХОДНИКОВ ----------
COPY . .

# ---------- СБОРКА ПРИЛОЖЕНИЯ (если нужно) ----------
# RUN npm run build

# ---------- НАСТРОЙКА ПРАВ ----------
RUN chown -R ${APP_USER}:${APP_USER} ${APP_HOME}

# ---------- ПЕРЕКЛЮЧЕНИЕ НА НЕ-ROOT ПОЛЬЗОВАТЕЛЯ ----------
USER ${APP_USER}

# ---------- HEALTHCHECK ----------
HEALTHCHECK --interval=30s --timeout=10s --start-period=40s --retries=3 \
  CMD curl -f http://localhost:3000/health || exit 1

# ---------- ПОРТЫ ----------
EXPOSE 3000

# ---------- КОМАНДА ЗАПУСКА ----------
CMD ["node", "server.js"]