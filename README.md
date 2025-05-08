
# Kittygram
![Workflow Status](https://github.com/Vasily-Sizov/kittygram_final/actions/workflows/main.yaml/badge.svg)

## Описание
**Kittygram** - это веб-приложение для любителей кошек, где пользователи могут делиться фотографиями своих питомцев, указывать их имя, цвет и год рождения, а также добавлять достижения.

---

## Технологии
**Backend**
- Python 3.9
- Django
- Django REST Framework
- PostgreSQL
- Gunicorn
- Python-dotenv
- Pillow

**Frontend**
- React
- Node.js 18

**Инфраструктура**
- NGINX
- Docker
- Docker Compose
- GitHub Actions (CI/CD)
- База данных: PostgreSQL 13
- Тестирование: Flake8, Django Test Framework, npm test

---

## Как запустить проект

**1. Клонирование репозитория**

- `git clone https://github.com/Vasily-Sizov/kittygram_final.git`
- `cd kittygram_final`

**2. Создание и настройка `.env` файла**

Создайте файл `.env` в корневой директории проекта и добавьте следующие переменные:

```
POSTGRES_DB=kittygram
POSTGRES_USER=kittygram_user
POSTGRES_PASSWORD=kittygram_password
DB_NAME=kittygram
DB_HOST=db
DB_PORT=5432

SECRET_KEY = your_secret_key
DEBUG=False
ALLOWED_HOSTS=your_domain_or_ip
USE_SQLITE=False
```

**3. Запуск проекта с помощью Docker Compose**

`docker-compose -f docker-compose.production.yml up -d`

**4. Применение миграций и сбор статических файлов**

```
docker-compose -f docker-compose.production.yml exec backend python manage.py migrate
docker-compose -f docker-compose.production.yml exec backend python manage.py collectstatic --noinput
```

---

## Общая схема работы

**1. DNS и внешняя сеть**
- Пользователь вводит `kittygram1.ru` в браузере
- Настройки DNS домена `kittygram1.ru` содержат А-запись, указывающую на IP-адрес `89.169.171.253`
- DNS-сервер преобразует доменное имя в IP-адрес
- Когда пользователь обращается к домену, его запрос фактически направляется на IP-адрес сервера
- Стандартный порт для HTTP-запросов: `80`

**2. Хост (внешний Nginx)**
- На сервере с IP `89.169.171.253` работает Nginx
- Конфигурация в `/etc/nginx/sites-enabled/default` содержит:

```nginx
server {
      server_name 89.169.171.253 kittygram1.ru;
      
      location / {
          proxy_set_header Host $http_host;
          proxy_pass http://127.0.0.1:9000;
      }
  }
```
- Nginx видит, что запрос предназначен для одного из его `server_name`
- Перенаправляет с порта `80` на порт `9000` хоста: `proxy_pass http://127.0.0.1:9000`

**3. Docker проброс**
- Настройка в `docker-compose.yml: ports: - 9000:80`
- Связывает порт `9000` хоста с портом `80` контейнера `gateway`

**4. Gateway (внутренний Nginx)**
- Контейнер gateway (Nginx) слушает порт `80`
- Конфигурация определяет маршрутизацию:
    - Для запросов`/api/` и `/admin/` → перенаправляет на `backend:9000`
    - Для `/media/` → обслуживает медиафайлы (загруженные пользователями изображения) из примонтированного тома `/media/`
    - Для остальных запросов → обслуживает статические файлы (HTML, CSS, JS, изображения) из тома `/static/`

**5. Тома для статики и медиа**
- В `docker-compose.yml` определены тома:
```yaml
volumes:
    static:
    media:
```
- Том `static` монтируется в:
    - контейнер `frontend` (для сборки статики)
    - контейнер `backend` (для сбора статики Django)
    - контейнер `gateway` (для раздачи статики)
- Том `media` монтируется в:
    - контейнер `backend` (для загрузки файлов)
    - контейнер `gateway` (для раздачи файлов)

**6. Backend**
- `Gunicorn` в контейнере `backend` слушает на порту `9000: --bind 0.0.0.0:9000`
- Запускается через команду в Dockerfile: `CMD ["gunicorn", "--bind", "0.0.0.0:9000", "kittygram_backend.wsgi"]`
- WSGI-файл `kittygram_backend.wsgi` содержит настройки запуска приложения Django
- Django обрабатывает динамические запросы и возвращает ответ обратно по той же цепочке

**7. База данных**
- Контейнер `db` с `PostgreSQL` хранит данные приложения
- Бэкенд соединяется с базой данных через внутреннюю Docker-сеть
- Данные сохраняются в томе `pg_data`

---

## docker-compose.yml

**Тома (volumes):**
- `pg_data` - для хранения данных PostgreSQL
- `static` - для статических файлов
- `media` - для медиафайлов

**Сервисы:**
1. `db` - база данных `PostgreSQL 13` 
- использует переменные среды из `.env` 
- хранит данные в томе `pg_data`
2. `backend` - серверная часть приложения:
- Собирается из директории `./backend/`
- Использует переменные среды из `.env`
- Монтирует тома для статических и медиафайлов
- Зависит от сервиса базы данных
3. `frontend` - клиентская часть:
- Собирается из директории `./frontend/`
- Использует переменные среды из `.env`
- Копирует собранные файлы в том `static`
4. gateway - Nginx в качестве прокси-сервера:
- Собирается из директории `./nginx/`
- Использует переменные среды из `.env`
- Монтирует тома для статических и медиафайлов
- Открывает порт 9000 (внешний) → 80 (внутренний)
- Зависит от сервиса `backend`

---

## CI/CD

1. Основные параметры
- Название: `Main Kittygram workflow`
- Триггер: Срабатывает при `push` в ветку `main`
2. Тестирование бэкенда (**backend_tests**)
- Запускается на Ubuntu
- Поднимает `PostgreSQL` в `Docker` для тестов
- Устанавливает `Python 3.9` и зависимости
- Запускает линтер `flake8`
- Запускает Django-тесты
3. Сборка и публикация Docker-образа бэкенда (**build_backend_and_push_to_docker_hub**)
- Зависит от успешного прохождения тестов бэкенда
- Собирает Docker-образ бэкенда из директории `./backend/`
- Публикует в DockerHub с тегом `vasilysizov1/kittygram_backend:latest`
4. Тестирование фронтенда (**frontend_tests**)
- Устанавливает `Node.js 18`
- Устанавливает зависимости через `npm`
- Запускает тесты фронтенда
5. Сборка и публикация Docker-образа фронтенда (**build_frontend_and_push_to_docker_hub**)
- Зависит от успешного прохождения тестов фронтенда
- Собирает Docker-образ фронтенда из директории `./frontend/`
- Публикует в DockerHub с тегом `vasilysizov1/kittygram_frontend:latest`
6. Сборка и публикация Docker-образа шлюза (**build_gateway_and_push_to_docker_hub**)
- Собирает Docker-образ Nginx (gateway) из директории `./gateway/`
- Публикует в DockerHub с тегом `vasilysizov1/kittygram_gateway:latest`
7. Деплой на сервер (**deploy**)
- Зависит от успешной сборки и публикации всех трех Docker-образов
- Выполняет действия:
    - 1. Копирует `docker-compose.production.yml` на сервер в директорию `kittygram`
    - 2. Подключается к серверу по `SSH`
    - 3. Выполняет следующие команды:
        - Переходит в директорию `kittygram`
        - Обновляет Docker-образы (`docker compose pull`)
        - Останавливает контейнеры (`down`)
        - Запускает контейнеры (`up -d`)
        - Выполняет миграции БД
        - Собирает статические файлы
        - Копирует собранную статику в нужную директорию

---

## Автор

- Сизов Василий