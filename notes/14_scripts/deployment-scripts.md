#!/bin/bash
# Скрипт для деплоя приложения через Docker
# Использование: ./deploy.sh [prod|staging|rollback]

set -e

# Конфигурация
APP_NAME="myapp"
DOCKER_REGISTRY=${DOCKER_REGISTRY:-"registry.example.com"}
ENVIRONMENT=${1:-"staging"}
TIMESTAMP=$(date +%Y%m%d_%H%M%S)

# Цвета
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
RED='\033[0;31m'
NC='\033[0m'

log() {
    echo -e "${GREEN}[$(date '+%Y-%m-%d %H:%M:%S')]${NC} $1"
}

error() {
    echo -e "${RED}[ERROR] $1${NC}" >&2
    exit 1
}

# Проверка переменных окружения
check_env() {
    if [ "$ENVIRONMENT" = "prod" ] || [ "$ENVIRONMENT" = "production" ]; then
        COMPOSE_FILE="docker-compose.prod.yml"
        ENV_FILE=".env.production"
        log "Deploying to PRODUCTION environment"
    elif [ "$ENVIRONMENT" = "staging" ]; then
        COMPOSE_FILE="docker-compose.staging.yml"
        ENV_FILE=".env.staging"
        log "Deploying to STAGING environment"
    elif [ "$ENVIRONMENT" = "rollback" ]; then
        log "Rolling back to previous version"
    else
        error "Unknown environment: $ENVIRONMENT. Use: prod, staging, rollback"
    fi
    
    if [ ! -f "$ENV_FILE" ] && [ "$ENVIRONMENT" != "rollback" ]; then
        error "Environment file $ENV_FILE not found!"
    fi
}

# Создание бэкапа перед деплоем
create_backup() {
    log "Creating backup of current deployment..."
    
    BACKUP_DIR="/backups/$APP_NAME/$TIMESTAMP"
    mkdir -p "$BACKUP_DIR"
    
    # Копируем текущие конфиги
    cp docker-compose*.yml "$BACKUP_DIR/" 2>/dev/null || true
    cp .env* "$BACKUP_DIR/" 2>/dev/null || true
    
    # Бэкап текущего состояния Docker
    docker-compose -f "$COMPOSE_FILE" config > "$BACKUP_DIR/docker-compose.current.yml"
    
    log "Backup saved to: $BACKUP_DIR"
}

# Pull последнего образа
pull_image() {
    log "Pulling latest image from registry..."
    
    if [ -n "$DOCKER_REGISTRY" ]; then
        docker pull "$DOCKER_REGISTRY/$APP_NAME:latest"
        export IMAGE_TAG="$DOCKER_REGISTRY/$APP_NAME:latest"
    fi
}

# Деплой
deploy() {
    log "Starting deployment..."
    
    # Остановка старых контейнеров
    docker-compose -f "$COMPOSE_FILE" down --remove-orphans || true
    
    # Запуск новых контейнеров
    docker-compose -f "$COMPOSE_FILE" up -d
    
    # Проверка здоровья
    log "Checking service health..."
    sleep 10
    
    if docker-compose -f "$COMPOSE_FILE" ps | grep -q "unhealthy"; then
        error "Some services are unhealthy! Rolling back..."
        rollback
    fi
    
    log "Deployment completed successfully"
}

# Проверка работоспособности
health_check() {
    log "Running health checks..."
    
    # Проверка через curl
    if command -v curl &> /dev/null; then
        for i in {1..5}; do
            if curl -s -f "http://localhost/health" > /dev/null; then
                log "Health check passed"
                return 0
            fi
            log "Waiting for service to be ready... ($i/5)"
            sleep 5
        done
        error "Health check failed!"
    fi
}

# Очистка старых образов
cleanup() {
    log "Cleaning up old images..."
    
    # Удаляем старые образы
    docker image prune -f
    
    # Удаляем старые тома (осторожно!)
    # docker volume prune -f
    
    log "Cleanup completed"
}

# Откат
rollback() {
    log "Performing rollback to previous version..."
    
    # Найти последний успешный бэкап
    LATEST_BACKUP=$(ls -td /backups/$APP_NAME/* | head -1)
    
    if [ -n "$LATEST_BACKUP" ] && [ -d "$LATEST_BACKUP" ]; then
        log "Restoring from backup: $LATEST_BACKUP"
        
        # Восстановить конфиги
        cp "$LATEST_BACKUP"/*.yml . 2>/dev/null || true
        cp "$LATEST_BACKUP"/.env* . 2>/dev/null || true
        
        # Перезапуск
        docker-compose -f "$COMPOSE_FILE" down
        docker-compose -f "$COMPOSE_FILE" up -d
        
        log "Rollback completed"
    else
        error "No backup found for rollback!"
    fi
}

# Отправка уведомления
notify() {
    local status=$1
    
    if [ -n "$SLACK_WEBHOOK" ]; then
        MESSAGE="Deployment to $ENVIRONMENT: $status"
        curl -s -X POST -H 'Content-type: application/json' \
            --data "{\"text\":\"$MESSAGE\"}" \
            "$SLACK_WEBHOOK" > /dev/null
    fi
}

# Основная функция
main() {
    log "Deployment script started for $APP_NAME"
    
    check_env
    
    if [ "$ENVIRONMENT" = "rollback" ]; then
        rollback
    else
        create_backup
        pull_image
        deploy
        health_check
        cleanup
        notify "success"
    fi
    
    log "Script completed"
}

main