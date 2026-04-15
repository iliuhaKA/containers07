# Лабораторная работа №7: Создание многоконтейнерного приложения

## Цель работы

Ознакомиться с работой многоконтейнерного приложения на базе `docker-compose`.

## Задание

Создать PHP приложение на базе трёх контейнеров: `nginx`, `php-fpm`, `mariadb`, используя `docker-compose`.

## Описание выполнения работы

### Шаг 1. Подготовка репозитория

Создан репозиторий `containers07` и склонирован на локальный компьютер. В корне репозитория подготовлена структура директорий для конфигурационных файлов и исходного кода сайта.

### Шаг 2. Размещение PHP-сайта

В директории `containers07` создана директория `mounts/site`, в которую скопирован сайт на PHP, разработанный в рамках предмета по PHP. Эта директория монтируется в контейнеры `frontend` (nginx) и `backend` (php-fpm), что обеспечивает доступ обоих сервисов к одному и тому же исходному коду.

### Шаг 3. Настройка `.gitignore`

В корне проекта создан файл [.gitignore](.gitignore), в который добавлены строки, исключающие содержимое директории сайта из отслеживания Git:

```gitignore
# Ignore files and directories
mounts/site/*
```

Это позволяет хранить конфигурацию инфраструктуры в репозитории, но не засорять его файлами самого сайта.

### Шаг 4. Конфигурация Nginx

Создан файл [nginx/default.conf](nginx/default.conf) со следующим содержимым:

```nginx
server {
    listen 80;
    server_name _;
    root /var/www/html;
    index index.php;
    location / {
        try_files $uri $uri/ /index.php?$args;
    }
    location ~ \.php$ {
        fastcgi_pass backend:9000;
        fastcgi_index index.php;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;
    }
}
```

Nginx слушает 80-й порт, отдаёт статические файлы напрямую, а PHP-запросы проксирует по FastCGI в сервис `backend` (php-fpm) на порт `9000`. Имя `backend` разрешается через внутреннюю сеть Docker Compose.

### Шаг 5. Описание сервисов в `docker-compose.yml`

Создан файл [docker-compose.yml](docker-compose.yml), описывающий три сервиса:

```yaml
version: '3.9'

services:
  frontend:
    image: nginx:1.19
    volumes:
      - ./mounts/site:/var/www/html
      - ./nginx/default.conf:/etc/nginx/conf.d/default.conf
    ports:
      - "80:80"
    networks:
      - internal
  backend:
    image: php:7.4-fpm
    volumes:
      - ./mounts/site:/var/www/html
    networks:
      - internal
    env_file:
      - mysql.env
  database:
    image: mysql:8.0
    env_file:
      - mysql.env
    networks:
      - internal
    volumes:
      - db_data:/var/lib/mysql

networks:
  internal: {}

volumes:
  db_data: {}
```

- `frontend` — контейнер Nginx, публикует порт 80 наружу.
- `backend` — контейнер PHP-FPM, обрабатывает PHP-скрипты.
- `database` — контейнер MySQL 8.0, хранит данные в именованном томе `db_data`.
- Все сервисы объединены в общую сеть `internal`, благодаря чему они видят друг друга по именам.

### Шаг 6. Переменные окружения MySQL

Создан файл [mysql.env](mysql.env) с параметрами подключения к БД:

```env
MYSQL_ROOT_PASSWORD=secret
MYSQL_DATABASE=app
MYSQL_USER=user
MYSQL_PASSWORD=secret
```

Этот файл подключается через директиву `env_file` к сервисам `backend` и `database`, что позволяет PHP-приложению и самой СУБД использовать одни и те же учётные данные.

### Шаг 7. Запуск и тестирование

Запуск всего стека выполняется одной командой из корня проекта:

```bash
docker-compose up -d
```

После старта в браузере по адресу `http://localhost` открывается PHP-сайт. При первом обращении может отобразиться дефолтная страница Nginx — в этом случае достаточно перезагрузить страницу.

## Ответы на вопросы

### 1. В каком порядке запускаются контейнеры?

В `docker-compose.yml` не заданы зависимости `depends_on`, поэтому явного порядка запуска нет — Docker Compose стартует сервисы параллельно. Порядок создания контейнеров определяется порядком их объявления в файле: сначала `frontend`, затем `backend`, затем `database`. Однако фактическая готовность каждого контейнера к работе наступает асинхронно и не гарантируется этим порядком.

### 2. Где хранятся данные базы данных?

Данные базы данных хранятся в именованном томе Docker `db_data`, смонтированном в `/var/lib/mysql` внутри контейнера `database`. Том объявлен в секции `volumes` файла `docker-compose.yml` и управляется самим Docker (физически располагается в его области хранения, например `/var/lib/docker/volumes/` в Linux или в виртуальном диске Docker Desktop в Windows). Благодаря этому данные сохраняются между перезапусками и удалением контейнера.

### 3. Как называются контейнеры проекта?

Docker Compose формирует имена контейнеров из имени проекта (по умолчанию — имени директории) и имени сервиса. Для проекта `containers07` контейнеры получают имена:

- `containers07-frontend-1`
- `containers07-backend-1`
- `containers07-database-1`

### 4. Как добавить файл `app.env` с переменной `APP_VERSION` для сервисов `backend` и `frontend`?

Необходимо создать в корне проекта файл `app.env`:

```env
APP_VERSION=1.0.0
```

И подключить его к обоим сервисам через директиву `env_file` в `docker-compose.yml`. Поскольку у `backend` уже есть `env_file`, новый файл добавляется в список; у `frontend` директива добавляется целиком:

```yaml
services:
  frontend:
    image: nginx:1.19
    volumes:
      - ./mounts/site:/var/www/html
      - ./nginx/default.conf:/etc/nginx/conf.d/default.conf
    ports:
      - "80:80"
    networks:
      - internal
    env_file:
      - app.env
  backend:
    image: php:7.4-fpm
    volumes:
      - ./mounts/site:/var/www/html
    networks:
      - internal
    env_file:
      - mysql.env
      - app.env
```

После изменения конфигурации стек пересоздаётся командой `docker-compose up -d`, и переменная `APP_VERSION` становится доступна внутри обоих контейнеров.

## Выводы

В ходе лабораторной работы был собран и запущен типовой LEMP-стек из трёх контейнеров (`nginx`, `php-fpm`, `mysql`), управляемых `docker-compose`. На практике показано, как декларативно описывать многосервисное приложение: сети обеспечивают взаимное обнаружение сервисов по именам, именованные тома сохраняют состояние базы данных между перезапусками, а файлы `env_file` централизуют управление конфигурацией. Такой подход позволяет быстро разворачивать воспроизводимое окружение для разработки и тестирования PHP-приложений.
