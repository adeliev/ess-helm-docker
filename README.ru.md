# Element Server Suite - Развертывание через Docker

Полное, готовое к production развертывание Matrix homeserver с использованием Docker Compose, включающее Synapse, Element Web, Coturn TURN сервер и Synapse Admin.

[English](README.md) | [Русская версия](#)

## Возможности

- **Synapse** - Matrix homeserver (последняя версия)
- **Element Web** - Современный веб-клиент Matrix
- **PostgreSQL 16** - Высокопроизводительная база данных
- **Coturn** - TURN/STUN сервер для VoIP звонков
- **Synapse Admin** - Веб-интерфейс администрирования
- **Nginx-ready** - Преднастроенные конфигурации reverse proxy
- **SSL/TLS** - Полная поддержка HTTPS с Let's Encrypt
- **Docker Compose** - Простое развертывание и управление

## Архитектура

```
┌─────────────────────────────────────────────────────────────┐
│                         Nginx Proxy                         │
│  (Обрабатывает SSL и маршрутизацию)                        │
└────────┬──────────────┬─────────────┬──────────────────────┘
         │              │             │
    Port 443        Port 443      Port 443
         │              │             │
   ┌─────▼─────┐  ┌────▼─────┐ ┌────▼─────┐
   │  Synapse  │  │ Element  │ │  Admin   │
   │   :8008   │  │  Web     │ │  :8091   │
   └─────┬─────┘  │  :8090   │ └──────────┘
         │        └──────────┘
    ┌────▼─────┐
    │PostgreSQL│       ┌──────────┐
    │  :5432   │       │ Coturn   │
    └──────────┘       │ :3479    │
                       └──────────┘
```

## Требования

- Docker Engine 20.10+
- Docker Compose 2.0+
- Доменное имя с настроенным DNS
- SSL/TLS сертификаты (рекомендуется Let's Encrypt)
- Минимум 2GB RAM, 2 CPU ядра
- 20GB дискового пространства

## Быстрый старт

### 1. Клонирование и настройка

\`\`\`bash
git clone <url-вашего-репозитория>
cd element

# Копируем шаблон окружения
cp .env.example .env

# Генерируем безопасные пароли
openssl rand -base64 32  # Для POSTGRES_PASSWORD
openssl rand -base64 64  # Для TURN_SHARED_SECRET

# Редактируем .env вашими значениями
nano .env
\`\`\`

### 2. Настройка Coturn

\`\`\`bash
# Копируем шаблон конфигурации Coturn
cp config/coturn/turnserver.conf.example config/coturn/turnserver.conf

# Редактируем и устанавливаем TURN секрет (такой же как в .env)
nano config/coturn/turnserver.conf
\`\`\`

Обновите эти значения:
- \`realm\` - Ваш домен (например, turn.yourdomain.com)
- \`server-name\` - То же что realm
- \`static-auth-secret\` - Такой же как TURN_SHARED_SECRET из .env
- \`external-ip\` - Публичный IP вашего сервера (опционально)

### 3. Начальная настройка Synapse

\`\`\`bash
# Генерируем конфигурацию Synapse
docker compose up synapse

# Дождитесь сообщения "Generating config file..."
# Затем остановите с помощью Ctrl+C
docker compose down
\`\`\`

### 4. Настройка Synapse

Отредактируйте \`data/synapse/homeserver.yaml\`:

\`\`\`yaml
# Сервер
server_name: "matrix.yourdomain.com"
public_baseurl: "https://matrix.yourdomain.com"

# База данных
database:
  name: psycopg2
  args:
    user: synapse
    password: "ВАШ_POSTGRES_PASSWORD"  # Из .env
    database: synapse
    host: postgres
    port: 5432

# TURN сервер
turn_uris:
  - "turn:matrix.yourdomain.com:3479?transport=tcp"
  - "turn:matrix.yourdomain.com:3479?transport=udp"
turn_shared_secret: "ВАШ_TURN_SHARED_SECRET"  # Из .env
turn_user_lifetime: 86400000
turn_allow_guests: true

# Регистрация
enable_registration: true
enable_registration_without_verification: true

# Медиа
max_upload_size: 50M
\`\`\`

### 5. Настройка Element Web

Отредактируйте \`config/element-web/config.json\`:

\`\`\`json
{
    "default_server_config": {
        "m.homeserver": {
            "base_url": "https://matrix.yourdomain.com",
            "server_name": "matrix.yourdomain.com"
        }
    }
}
\`\`\`

### 6. Настройка Nginx

Скопируйте nginx конфигурации на ваш nginx сервер:

\`\`\`bash
# На вашем nginx сервере
sudo cp config/nginx/*.conf /etc/nginx/sites-available/
sudo ln -s /etc/nginx/sites-available/matrix.yourdomain.com.conf /etc/nginx/sites-enabled/
sudo ln -s /etc/nginx/sites-available/element.yourdomain.com.conf /etc/nginx/sites-enabled/

# Проверка и перезагрузка
sudo nginx -t
sudo systemctl reload nginx
\`\`\`

**Важно:** Обновите следующее в nginx конфигах:
- Замените `example.com` на ваш домен
- Обновите пути к SSL сертификатам
- Обновите IP в `proxy_pass` на IP вашего Docker хоста

### 7. Запуск всех сервисов

\`\`\`bash
docker compose up -d
\`\`\`

### 8. Создание администратора

\`\`\`bash
docker exec -it element-synapse register_new_matrix_user \\
  http://localhost:8008 -c /data/homeserver.yaml -a
\`\`\`

Следуйте подсказкам для создания учетной записи администратора.

## URL сервисов

После развертывания сервисы будут доступны по адресам:

- **Element Web:** https://element.yourdomain.com
- **Matrix API:** https://matrix.yourdomain.com
- **Synapse Admin:** http://ip-вашего-сервера:8091 (только локально)
- **Federation:** https://matrix.yourdomain.com:8448

## Порты

### Docker контейнеры
- \`8008\` - Synapse HTTP API
- \`8448\` - Synapse Federation
- \`8090\` - Element Web
- \`8091\` - Synapse Admin (только локально)
- \`3479\` - Coturn TURN/STUN
- \`5349\` - Coturn TLS
- \`49152-49252\` - Диапазон UDP relay Coturn

### Требования к файерволу
Откройте эти порты на вашем файерволе:
- \`80/tcp\` - HTTP (перенаправление на HTTPS)
- \`443/tcp\` - HTTPS
- \`8448/tcp\` - Matrix Federation
- \`3479/tcp,udp\` - TURN/STUN
- \`5349/tcp\` - TURN/STUN TLS (опционально)

## Управление

### Команды Docker Compose

\`\`\`bash
# Запуск всех сервисов
docker compose up -d

# Остановка всех сервисов
docker compose down

# Просмотр логов
docker compose logs -f

# Просмотр логов конкретного сервиса
docker compose logs -f synapse

# Перезапуск сервиса
docker compose restart synapse

# Обновление образов
docker compose pull
docker compose up -d
\`\`\`

### Управление пользователями

**Через Synapse Admin (рекомендуется):**
1. Откройте http://ip-вашего-сервера:8091
2. Войдите с учетными данными администратора
3. Управляйте пользователями, комнатами и настройками через веб-интерфейс

**Через командную строку:**
\`\`\`bash
# Создание пользователя
docker exec -it element-synapse register_new_matrix_user \\
  http://localhost:8008 -c /data/homeserver.yaml

# Сделать пользователя администратором
docker exec -it element-synapse \\
  sqlite3 /data/homeserver.db \\
  "UPDATE users SET admin = 1 WHERE name = '@username:yourdomain.com'"
\`\`\`

## Резервное копирование

### Важные данные

Всегда делайте резервные копии этих директорий:
- \`data/postgres/\` - База данных
- \`data/synapse/\` - Конфигурация homeserver и медиа
- \`config/\` - Все конфигурации

### Скрипт резервного копирования

\`\`\`bash
#!/bin/bash
BACKUP_DIR="/backups/element"
DATE=$(date +%Y%m%d)

# Остановка сервисов
docker compose down

# Создание резервной копии
tar -czf "$BACKUP_DIR/element-backup-$DATE.tar.gz" \\
  data/ config/ docker-compose.yml .env

# Перезапуск сервисов
docker compose up -d
\`\`\`

### Восстановление

\`\`\`bash
# Остановка сервисов
docker compose down

# Извлечение резервной копии
tar -xzf element-backup-YYYYMMDD.tar.gz

# Запуск сервисов
docker compose up -d
\`\`\`

## Устранение неполадок

### Synapse не запускается

Проверьте логи:
\`\`\`bash
docker compose logs synapse
\`\`\`

Частые проблемы:
- Не удалось подключиться к БД → Проверьте POSTGRES_PASSWORD в .env и homeserver.yaml
- Порт уже используется → Проверьте, не использует ли другой сервис порт 8008
- Ошибки в конфигурации → Проверьте YAML синтаксис в homeserver.yaml

### Голосовые/видео звонки не работают

1. Проверьте что Coturn запущен:
\`\`\`bash
docker compose ps coturn
\`\`\`

2. Убедитесь что TURN секрет совпадает в:
   - \`.env\` (TURN_SHARED_SECRET)
   - \`config/coturn/turnserver.conf\` (static-auth-secret)
   - \`data/synapse/homeserver.yaml\` (turn_shared_secret)

3. Проверьте TURN сервер:
\`\`\`bash
# Проверка открыт ли порт
nc -zv ip-вашего-сервера 3479
\`\`\`

### Федерация не работает

1. Протестируйте федерацию:
   - Посетите https://federationtester.matrix.org/
   - Введите домен вашего homeserver

2. Проверьте доступность порта 8448:
\`\`\`bash
curl https://matrix.yourdomain.com:8448/_matrix/federation/v1/version
\`\`\`

3. Проверьте \`.well-known\` эндпоинты:
\`\`\`bash
curl https://matrix.yourdomain.com/.well-known/matrix/server
curl https://matrix.yourdomain.com/.well-known/matrix/client
\`\`\`

## Безопасность

1. **Используйте надежные пароли**
   - Генерируйте с помощью \`openssl rand -base64 32\`
   - Никогда не коммитьте файл \`.env\` в git

2. **Ограничьте регистрацию**
   - Установите \`enable_registration: false\` после создания пользователей
   - Или используйте \`registrations_require_3pid\` для верификации email

3. **Регулярные обновления**
   \`\`\`bash
   docker compose pull
   docker compose up -d
   \`\`\`

4. **Настройка файервола**
   - Открывайте только необходимые порты
   - Используйте fail2ban для защиты от brute force

5. **SSL/TLS**
   - Всегда используйте HTTPS
   - Обновляйте сертификаты
   - Используйте сильные шифры (уже настроено в nginx)

6. **Резервные копии**
   - Автоматизируйте регулярные бэкапы
   - Тестируйте процедуры восстановления
   - Храните бэкапы в безопасном месте

## Мониторинг

### Проверки работоспособности

\`\`\`bash
# Synapse health
curl http://localhost:8008/health

# Element Web
curl http://localhost:8090

# PostgreSQL
docker exec element-postgres pg_isready -U synapse
\`\`\`

### Использование ресурсов

\`\`\`bash
# Статистика контейнеров
docker stats

# Использование диска
du -sh data/
\`\`\`

## Обновление

\`\`\`bash
# Загрузка последних образов
docker compose pull

# Остановка сервисов
docker compose down

# Резервная копия перед обновлением (рекомендуется)
tar -czf backup-pre-upgrade.tar.gz data/ config/

# Запуск с новыми образами
docker compose up -d

# Проверка логов на ошибки
docker compose logs -f
\`\`\`

## Поддержка

- [Matrix Specification](https://spec.matrix.org/)
- [Synapse Documentation](https://element-hq.github.io/synapse/)
- [Element Documentation](https://element.io/help)
- [Coturn Documentation](https://github.com/coturn/coturn/wiki)

## Благодарности

Основано на официальных [Element Server Suite Helm Charts](https://github.com/element-hq/ess-helm), адаптированных для развертывания через Docker Compose.
