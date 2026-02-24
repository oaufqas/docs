# Шаблон многоступенчатой сборки для React + Node.js
# Frontend: React, Backend: Node.js Express
# Результат: один образ с сервером и статикой

# ---- Stage 1: Build React ----
FROM node:18-alpine AS frontend-builder

WORKDIR /build

# Копируем package.json фронтенда
COPY frontend/package*.json ./

# Устанавливаем зависимости
RUN npm ci

# Копируем исходники фронтенда
COPY frontend/ ./

# Собираем React приложение
RUN npm run build

# ---- Stage 2: Setup Backend ----
FROM node:18-alpine AS backend-builder

WORKDIR /build

# Копируем package.json бэкенда
COPY backend/package*.json ./

# Устанавливаем только production зависимости
RUN npm ci --only=production

# ---- Stage 3: Final Image ----
FROM node:18-alpine

# Создаем пользователя
RUN addgroup -g 1001 -S nodejs && \
    adduser -S nodejs -u 1001

WORKDIR /app

# Копируем зависимости бэкенда
COPY --from=backend-builder --chown=nodejs:nodejs /build/node_modules ./node_modules

# Копируем код бэкенда
COPY --chown=nodejs:nodejs backend/ ./

# Копируем собранный фронтенд в папку public
COPY --from=frontend-builder --chown=nodejs:nodejs /build/build ./public

# Переключаемся на непривилегированного пользователя
USER nodejs

# Порт
EXPOSE 5000

# Healthcheck
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD node -e "require('http').get('http://localhost:5000/health', (r) => {process.exit(r.statusCode === 200 ? 0 : 1)})"

# Запуск
CMD ["node", "server.js"]