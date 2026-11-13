# automated-backup-archiving-script
#!/bin/bash
# ==============================================================================
# Script Name: backup.sh
# Description: Automated Backup & Archiving Script for Linux Systems
# Features:
#   - Creates compressed, timestamped tarball archives (.tar.gz)
#   - Validates target source and destination directories
#   - Automatically purges backups older than the retention threshold
#   - Logs all operations to a system log file
# ==============================================================================

set -euo pipefail

# ----------------- Configuration Settings -----------------
# Directory to back up (e.g., website files, databases, or documents)
SOURCE_DIR="${SOURCE_DIR:-/var/www/html}"

# Directory where archives will be stored
BACKUP_DIR="${BACKUP_DIR:-/backups}"

# System log file path
LOG_FILE="${LOG_FILE:-/var/log/backup.log}"

# Retention period in days (older archives are automatically purged)
RETENTION_DAYS="${RETENTION_DAYS:-7}"

# Timestamp formatting (YYYY-MM-DD_HHMMSS)
TIMESTAMP=$(date +"%Y-%m-%d_%H%M%S")
BACKUP_FILE="$BACKUP_DIR/backup_$TIMESTAMP.tar.gz"

# ----------------- Helper Functions -----------------
log_message() {
    local level="$1"
    local message="$2"
    local entry="[$(date '+%Y-%m-%d %H:%M:%S')] [$level] $message"
    echo "$entry"
    if [ -w "$(dirname "$LOG_FILE")" ] || [ -w "$LOG_FILE" ]; then
        echo "$entry" >> "$LOG_FILE"
    fi
}

# ----------------- Execution Steps -----------------

# 1. Validation
log_message "INFO" "Starting backup process..."

if [ ! -d "$SOURCE_DIR" ]; then
    log_message "ERROR" "Source path '$SOURCE_DIR' does not exist. Aborting."
    exit 1
fi

if [ ! -d "$BACKUP_DIR" ]; then
    log_message "INFO" "Creating target backup directory: $BACKUP_DIR"
    mkdir -p "$BACKUP_DIR"
fi

# 2. Archive & Compression
log_message "INFO" "Compressing '$SOURCE_DIR' into '$BACKUP_FILE'..."

if tar -czf "$BACKUP_FILE" -C "$(dirname "$SOURCE_DIR")" "$(basename "$SOURCE_DIR")" 2>/dev/null; then
    BACKUP_SIZE=$(du -h "$BACKUP_FILE" | cut -f1)
    log_message "SUCCESS" "Backup created: $BACKUP_FILE (Size: $BACKUP_SIZE)"
else
    log_message "ERROR" "Failed to archive directory '$SOURCE_DIR'."
    exit 1
fi

# 3. Retention Cleanup
log_message "INFO" "Scanning for archives older than $RETENTION_DAYS days in $BACKUP_DIR..."

DELETED_COUNT=$(find "$BACKUP_DIR" -type f -name "backup_*.tar.gz" -mtime +"$RETENTION_DAYS" | wc -l)
find "$BACKUP_DIR" -type f -name "backup_*.tar.gz" -mtime +"$RETENTION_DAYS" -exec rm -f {} \;

log_message "INFO" "Cleanup complete. Removed $DELETED_COUNT expired archive(s)."
log_message "SUCCESS" "Backup job completed."
exit 0
