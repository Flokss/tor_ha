## Суть проблемы
Апстрим-аддон `alexbelgium/tor-amd64` при каждом запуске дописывает (>>) конфиг в `/etc/tor/torrc`.
При рестарте появляются дубликаты блоков `HiddenServiceDir`. Tor 0.4.8.14 падает с ошибкой парсинга,
срабатывает watchdog и аддон уходит в бесконечный цикл крашей.
Ручное удаление контейнера помогает временно, пока не произойдет следующий рестарт.

## Решение
Собран патченный локальный аддон в папке `C:\Users\admin\AppData\Local\Temp\opencode\tor_addon\`.
Патч: скрипт `init-tor/run` сначала сбрасывает `torrc` в исходное состояние (`torrc.sample`),
затем дописывает конфиг. Ключи из `/ssl/tor/hidden_service/` сохраняются.

## Как собрать и установить на другой машине (где есть Docker)

### 1. Перенесите файлы
Скопируйте папку `tor_addon` целиком на хост с Docker.

### 2. Соберите образ
В терминале в папке `tor_addon` выполните:
```bash
docker build -t tor-fixed:5.0.3-patch1 .
```

### 3. Перенесите образ в Home Assistant
Если HA работает в Docker/Synology/Proxmox:
```bash
# Вариант А: через save/load
docker save tor-fixed:5.0.3-patch1 | bzip2 > tor-fixed.tar.bz2
# Скопируйте tor-fixed.tar.bz2 на хост HA
# На хосте HA:
docker load < tor-fixed.tar.bz2
```
Если HA Supervisor:
```bash
# Вариант Б: загрузите в локальный registry или используйте Supervisor API
# Самый простой способ для Supervisor:
# Положите файлы в /addons/tor_fixed/ на хосте HA
# В HA: Settings -> Add-ons -> Store -> ⋮ -> Repository -> https://github.com/user/repo
# Или просто "Install" из локальной папки через Supervisor CLI:
ha addons rebuild tor_fixed
ha addons start tor_fixed
```

### 4. Настройка аддона в HA
1. Settings -> Add-ons -> "Tor with bridges (fixed)" -> Configuration
2. Убедитесь что маппинг `ssl:rw` активен (ключи `/ssl/tor/hidden_service/` должны сохраниться)
3. Start -> Check logs. Должно быть: `Tor init complete` без ошибок парсинга.
4. Проверьте `.onion` адрес в `/ssl/tor/hidden_service/hostname`

## Примечания
- Архитектура: `amd64` (совпадает с апстримом)
- Сеть: `host_network: true` (требуется оригинальному аддону)
- Маппинг: `ssl:rw` обязателен для сохранения ключей
- Если используете мосты (bridges), добавьте их в Options аддона в формате строки
