#!/bin/bash
# Скрипт для бэкапа баз данных и файлов
# Использование: ./backup.sh [db|files|all]

set -e  # Выход при ошибке

# Цвета для вывода
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m' # No Color

# Конфигурация
BACKUP_DIR="/var/backups/$(date +%Y%m%d)"
DB_NAME=${DB_NAME:-"myapp"}
DB_USER=${DB_USER:-"postgres"}
DB_PASS=${DB_PASS:-""}
DB_HOST=${DB_HOST:-"localhost"}
RETENTION_DAYS=7
S3_BUCKET=${S3_BUCKET:-"s3://myapp-backups"}

# Функция логирования
log() {
    echo -e "${GREEN}[$(date '+%Y-%m-%d %H:%M:%S')]${NC} $1"
}

error() {
    echo -e "${RED}[ERROR] $1${NC}" >&2
    exit 1
}

warn() {
    echo -e "${YELLOW}[WARN] $1${NC}"
}

# Создание директории для бэкапов
create_backup_dir() {
    mkdir -p "$BACKUP_DIR"
    log "Created backup directory: $BACKUP_DIR"
}

# Бэкап PostgreSQL
backup_postgres() {
    log "Starting PostgreSQL backup..."
    
    if [ -z "$DB_PASS" ]; then
        export PGPASSWORD="$DB_PASS"
    fi
    
    BACKUP_FILE="$BACKUP_DIR/postgres_$DB_NAME.sql.gz"
    
    pg_dump -h "$DB_HOST" -U "$DB_USER" "$DB_NAME" | gzip > "$BACKUP_FILE"
    
    if [ $? -eq 0 ] && [ -f "$BACKUP_FILE" ]; then
        SIZE=$(du -h "$BACKUP_FILE" | cut -f1)
        log "PostgreSQL backup completed: $BACKUP_FILE ($SIZE)"
    else
        error "PostgreSQL backup failed!"
    fi
}

# Бэкап MySQL/MariaDB
backup_mysql() {
    log "Starting MySQL backup..."
    
    BACKUP_FILE="$BACKUP_DIR/mysql_$DB_NAME.sql.gz"
    
    mysqldump -h "$DB_HOST" -u "$DB_USER" -p"$DB_PASS" "$DB_NAME" | gzip > "$BACKUP_FILE"
    
    if [ $? -eq 0 ] && [ -f "$BACKUP_FILE" ]; then
        SIZE=$(du -h "$BACKUP_FILE" | cut -f1)
        log "MySQL backup completed: $BACKUP_FILE ($SIZE)"
    else
        error "MySQL backup failed!"
    fi
}

# Бэкап файлов
backup_files() {
    log "Starting files backup..."
    
    SOURCE_DIR=${1:-"/var/www"}
    BACKUP_FILE="$BACKUP_DIR/files_backup.tar.gz"
    
    tar -czf "$BACKUP_FILE" -C "$SOURCE_DIR" .
    
    if [ $? -eq 0 ] && [ -f "$BACKUP_FILE" ]; then
        SIZE=$(du -h "$BACKUP_FILE" | cut -f1)
        log "Files backup completed: $BACKUP_FILE ($SIZE)"
    else
        error "Files backup failed!"
    fi
}

# Очистка старых бэкапов
cleanup_old_backups() {
    log "Cleaning backups older than $RETENTION_DAYS days..."
    
    find /var/backups -type d -mtime +$RETENTION_DAYS -exec rm -rf {} \; 2>/dev/null || true
    
    log "Cleanup completed"
}

# Загрузка в S3 (опционально)
upload_to_s3() {
    if [ -n "$S3_BUCKET" ] && command -v aws &> /dev/null; then
        log "Uploading to S3: $S3_BUCKET"
        aws s3 sync /var/backups "$S3_BUCKET/$(date +%Y%m%d)" --delete
        log "S3 upload completed"
    else
        warn "AWS CLI not found or S3 bucket not configured, skipping upload"
    fi
}

# Отправка уведомления (опционально)
send_notification() {
    local status=$1
    local message=$2
    
    if [ -n "$SLACK_WEBHOOK" ]; then
        curl -s -X POST -H 'Content-type: application/json' \
            --data "{\"text\":\"Backup $status: $message\"}" \
            "$SLACK_WEBHOOK" > /dev/null
    fi
}

# Основная функция
main() {
    log "Starting backup process..."
    
    create_backup_dir
    
    case "${1:-all}" in
        postgres|pg)
            backup_postgres
            ;;
        mysql)
            backup_mysql
            ;;
        files)
            backup_files
            ;;
        all)
            backup_postgres
            backup_mysql
            backup_files
            ;;
        *)
            error "Unknown option: $1. Use: postgres, mysql, files, all"
            ;;
    esac
    
    cleanup_old_backups
    upload_to_s3
    send_notification "success" "Backup completed successfully"
    
    log "Backup process completed!"
}

# Запуск
main "$@"