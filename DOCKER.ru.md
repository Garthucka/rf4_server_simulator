# Развёртывание в одном контейнере Docker

`Dockerfile` в корне репозитория упаковывает `seed`, `gameserver`, `gateway` и отдельную веб-админку в один образ. Внутри контейнера работает пользователь без прав root. HTTPS-шлюз слушает порт `8443`, а хост отдаёт его наружу как `443` через проброс портов.

## Сборка

Выполните в корне репозитория:

```sh
docker build \
  --build-arg GIT_COMMIT="$(git rev-parse --short HEAD)" \
  --build-arg SOURCE_DIR="$PWD" \
  -t rf4-server-single .
```

Значения `VERSION` / `BUILD_TS` по умолчанию формируются из времени сборки. Можно передать явно метку времени UTC `BUILD_TS=yyyyMMddHHmmss` и соответствующий `VERSION`. Эти значения попадают в метрику `rf4_build_info` каждого процесса.

## Запуск

Обязательно нужно задать такие переменные окружения:

- `RF4_GATEWAY_ADVERTISE_HOST`: IPv4-адрес этой машины или локальной сети, который клиент использует для realtime-соединения.
- `RF4_ADMIN_PASSWORD`: пароль администратора веб-админки.
- `RF4_ADMIN_JWT_SECRET`: ключ подписи JWT для админки.
- `RF4_TUNNEL_PSK` и `RF4_GAME_TUNNEL_PSK`: значения должны совпадать.

```sh
docker run -d --name rf4-server-single \
  -e RF4_GATEWAY_ADVERTISE_HOST=127.0.0.1 \
  -e RF4_ADMIN_PASSWORD='replace-with-strong-password' \
  -e RF4_ADMIN_JWT_SECRET='replace-with-long-random-secret' \
  -e RF4_TUNNEL_PSK='replace-with-shared-random-secret' \
  -e RF4_GAME_TUNNEL_PSK='replace-with-shared-random-secret' \
  -p 127.0.0.1:443:8443 \
  -p 127.0.0.1:7515:7515 \
  -p 127.0.0.1:9341:9341 \
  -p 127.0.0.1:8080:8080 \
  -v rf4-player-data:/app/data \
  -v rf4-gateway-certs:/app/certs \
  rf4-server-single
```

При старте контейнер сначала запускает `seed`. После успеха он запускает gameserver, затем gateway и admin. Встроенная проверка работоспособности одновременно проверяет `/readyz` у трёх сервисов. Подключайте клиент, когда `docker ps` покажет `healthy`.

Админка доступна по адресу `http://127.0.0.1:8080/`, имя пользователя по умолчанию `admin`. Шаги подключения клиента, настройки hosts и доверия к сертификату описаны в [руководстве по развёртыванию](docs/deploy.ru.md#подключение-клиента). Скопируйте сертификат шлюза:

```sh
docker cp rf4-server-single:/app/certs/server.crt ./server.crt
```

База игроков и HTTPS-сертификаты хранятся в именованных томах `rf4-player-data` и `rf4-gateway-certs`. Удаление контейнера не удаляет сохранения. Они очищаются только командой `docker volume rm rf4-player-data rf4-gateway-certs`.
