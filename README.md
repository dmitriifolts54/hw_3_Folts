# ДЗ №3: Docker

[Условие задания](https://github.com/Nikfilk2030/missing_semester_hse_cs/blob/main/2026/amcs/year_1/hw_3/README.md)

Тексты файлов Dockerfile, .env, .env.example, compose.yaml, compose.sqlite.yaml, .dockerignore лежат в папке hw_3_Folts рядом с данным README.md

## 1. Один контейнер

### Сборка образа

```bash
sudo docker build -t counter-app .
```

### Запуск контейнера

```bash
sudo docker run -d \
  --name hw3-app \
  -p 127.0.0.1:8080:8000 \
  -e "GREETING=Hello, Docker!" \
  counter-app
```

### Проверка счётчика

```bash
curl -X GET http://127.0.0.1:8080/
```

```json
{"greeting": "Hello, Docker!", "count": 0, "hostname": "7174fc8c602a", "storage": "sqlite"}
```

```bash
curl -X POST http://127.0.0.1:8080/increment
```

```json
{"greeting": "Hello, Docker!", "count": 1, "hostname": "7174fc8c602a", "storage": "sqlite"}
```

```bash
curl -X GET http://127.0.0.1:8080/
```

```json
{"greeting": "Hello, Docker!", "count": 1, "hostname": "7174fc8c602a", "storage": "sqlite"}
```

Видим, что вызов `GET` не меняет counter, а `POST` - меняет. Этого мы и хотели

### Логи приложения

```bash
sudo docker logs hw3-app
```

```text
2026-11-24 07:47:21,821 listening port=8000 storage=sqlite
2026-11-24 07:48:44,838 http 172.17.0.1 "GET / HTTP/1.1" 200 -
2026-11-24 07:48:44,864 http 172.17.0.1 "POST /increment HTTP/1.1" 200 -
2026-11-24 07:48:44,877 http 172.17.0.1 "GET / HTTP/1.1" 200 -
```

Перед переходом к следующему пункту останавливаем контейнер

```bash
sudo docker stop hw3-app
```

## 2. Docker Compose

Для повторения этого уровня надо использовать `compose.sqlite.yaml`

### Запуск и настройки

```bash
cp .env.example .env
```

```bash
sudo docker compose -f compose.sqlite.yaml up -d --build
```

База `/data/counter.db` хранится в именованном volume `counter_data`, подключённом к `/data`

### Проверка сохранения данных

```bash
curl -X GET http://127.0.0.1:8080/
```

```json
{"greeting": "Hello, Docker!", "count": 0, "hostname": "fa613d119fcb", "storage": "sqlite"}
```


```bash
curl -X POST http://127.0.0.1:8080/increment
curl -X POST http://127.0.0.1:8080/increment
curl -X POST http://127.0.0.1:8080/increment
curl -X GET http://127.0.0.1:8080/
```

```json
{"greeting": "Hello, Docker!", "count": 3, "hostname": "fa613d119fcb", "storage": "sqlite"}
```

Пересоздание контейнера

```bash
sudo docker compose -f compose.sqlite.yaml down
sudo docker compose -f compose.sqlite.yaml up -d
curl -X GET http://127.0.0.1:8080/
```

```json
{"greeting": "Hello, Docker!", "count": 3, "hostname": "eb9924c39e5c", "storage": "sqlite"}
```

Видим, что `count` в новом контейнере сохранился, это и была наша цель

Логи после повторного запуска

```bash
sudo docker compose -f compose.sqlite.yaml logs app
```

```text
app-1  | 2026-11-24 08:14:56,458 listening port=8000 storage=sqlite
app-1  | 2026-11-24 08:14:56,774 http 172.18.0.1 "GET / HTTP/1.1" 200 -
```

Перед переходом к следующему пункту останавливаем SQLite-вариант

```bash
sudo docker compose -f compose.sqlite.yaml down
```

## 3. Redis 

### Запуск и настройки

```bash
sudo docker compose up -d --build
sudo docker compose ps
```

Оба сервиса выдают `healthy`, продолжаем

### Сохранение данных

Изначально в этой проверке значение `count` было 3. Три раза применив `POST`, увеличили его до 6

```bash
curl -X POST http://127.0.0.1:8080/increment
curl -X POST http://127.0.0.1:8080/increment
curl -X POST http://127.0.0.1:8080/increment
curl -X GET http://127.0.0.1:8080/
```

```json
{"greeting": "Hello, Docker!", "count": 6, "hostname": "e0cb62f50a48", "storage": "redis"}
```

```bash
sudo docker compose down
sudo docker compose up -d
sudo docker compose ps
```

```bash
curl -X GET http://127.0.0.1:8080/
```

```json
{"greeting": "Hello, Docker!", "count": 6, "hostname": "f2307b302ebe", "storage": "redis"}
```

### Отключение Redis

```bash
sudo docker compose stop redis

curl -i http://127.0.0.1:8080/healthz
curl -i http://127.0.0.1:8080/readyz
curl -i -X POST http://127.0.0.1:8080/increment
```

Вывод первой проверки

```text
HTTP/1.0 200 OK
Server: BaseHTTP/0.6 Python/3.13.18
Date: Tue, 24 Nov 2026 19:45:15 GMT
Content-Type: application/json; charset=utf-8
Content-Length: 19
Cache-Control: no-store
{"status": "alive"}
```

Вывод второй проверки

```text
HTTP/1.0 503 Service Unavailable
Server: BaseHTTP/0.6 Python/3.13.18
Date: Tue, 24 Nov 2026 19:46:09 GMT
Content-Type: application/json; charset=utf-8
Content-Length: 32
Cache-Control: no-store
{"error": "storage_unavailable"}
```

Вывод третьей проверки

```text
HTTP/1.0 503 Service Unavailable
Server: BaseHTTP/0.6 Python/3.13.18
Date: Tue, 24 Nov 2026 19:46:46 GMT
Content-Type: application/json; charset=utf-8
Content-Length: 32
Cache-Control: no-store
{"error": "storage_unavailable"}
```

### Восстановление Redis

```bash
sudo docker compose start redis
sudo docker compose ps
```

Видим такой вывод

```text
NAME                 IMAGE              COMMAND                  SERVICE   CREATED         STATUS                        PORTS
hw_3_folts-app-1     counter-app        "python app.py"          app       7 minutes ago   Up 7 minutes (healthy)        127.0.0.1:8080->8000/tcp
hw_3_folts-redis-1   redis:7.4-alpine   "docker-entrypoint.s…"   redis     7 minutes ago   Up About a minute (healthy)   6379/tcp
```

Проверяем, что всё работает правильно

```bash
curl -X GET http://127.0.0.1:8080/
```

```json
{"greeting": "Hello, Docker!", "count": 6, "hostname": "f2307b302ebe", "storage": "redis"}
```

```bash
curl -X POST http://127.0.0.1:8080/increment
```

```json
{"greeting": "Hello, Docker!", "count": 7, "hostname": "f2307b302ebe", "storage": "redis"}
```

Видно, что при перезапусках счётчик восстанавливается, что и хотели. Финальный статус обоих сервисов `healthy`
