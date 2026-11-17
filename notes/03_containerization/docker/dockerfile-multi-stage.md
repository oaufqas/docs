**Multi-stage builds** — это техника написания Dockerfile, которая позволяет сделать итоговый образ максимально легким (в 10-50 раз меньше оригинала), оставляя в нем только готовый скомпилированный код без «мусора» вроде компиляторов, исходников и кэша инструментов сборки.

Чтобы собрать приложение (например, на Go, Java или React), вам нужны инструменты: компиляторы, npm, maven, gcc. Все они весят сотни мегабайт. Если вы оставите их в образе, он будет огромным, медленным и небезопасным.

Вы **используете инструкцию FROM несколько раз внутри одного Dockerfile**. Каждый блок FROM — это новый этап (stage). Вы можете копировать файлы из одного этапа в другой, а в финальный образ попадет только то, что вы перенесли в последний блок.

#### Пример:

```yaml
# ЭТАП 1: Сборка (называем его "builder")
FROM golang:1.21-alpine AS builder
WORKDIR /app
COPY . .
# Компилируем бинарный файл
RUN go build -o myapp main.go

# ЭТАП 2: Финальный образ (минимальный)
FROM alpine:latest
WORKDIR /root/
# Копируем ТОЛЬКО один исполняемый файл из первого этапа
COPY --from=builder /app/myapp .

# Запускаем
CMD ["./myapp"]
```

**Результат:**
- Этап builder весил ~300 МБ.
- Итоговый образ на alpine весит ~10 МБ.
- Итог: Экономия места и безопасности (в финальном образе нет даже компилятора Go).

Вы можете копировать файлы даже не из своих этапов, а из любых готовых образов:

`COPY --from=nginx:latest /etc/nginx/nginx.conf /my/local/path`


---

```yml
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
```