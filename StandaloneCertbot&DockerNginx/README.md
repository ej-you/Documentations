# Запуск Nginx (в докере) в связке с Certbot (на сервере) для получения бесплатных SSL

> ПОДСКАЗКА: Используйте `sudo` если текущий пользователь не `root`

## Настройка Nginx

### docker-compose.yml

Для запуска у вас должен быть свой `docker-compose.yml` файл с необходимой конфигурацией.
В файле [docker-compose.yml][1] есть базовый пример.
___Особое внимание___ обратите на `volume` nginx сервиса. Эти тома важны для корректной работы certbot.

### web-server сервис

Для `web-server` сервиса docker можете использовать настройки из папки [web-server][2]
В этой папке лежит [Dockerfile][3] со сборкой фронта и запуском nginx, [главный конфиг][4] nginx и два конфига для настройки обслуживания 80 и 443 портов.
___Особое внимание___ обратите на то, что в [главном конфиге][4] использование [конфига][5] для обслуживания 443 порта закомментировано (в самом низу файла).

## Запуск Nginx

Запустите ваш `docker-compose.yml`, проверьте, что все контейнеры запущены. Например так:

```shell
docker compose up --build -d
docker compose ps
```

## Настройка certbot и выдача первого сертификата

Установите certbot:

```shell
apt update
apt install certbot
```

Создайте директорию для конфигурационных файлов certbot:

```shell
mkdir /var/www/certbot
chown www-data:www-data /var/www/certbot/
chmod 755 /var/www/certbot/
```

Зарегистрируйте аккант (если требуется):

```shell
certbot register -m your.email@gmail.com
```

Проверьте всё, выпустив первый сертификат тестово:

```shell
certbot certonly --webroot --webroot-path=/var/www/certbot/ -d domain.ru -d www.domain.ru --dry-run
```

Если всё прошло успешно, то выпустите первый сертификат:

```shell
certbot certonly --webroot --webroot-path=/var/www/certbot/ -d domain.ru -d www.domain.ru
```

## Корректировка Nginx

После успешного выпуска сертификата вы можете раскомментировать строку импорта [конфига][5] для обслуживания 443 порта в [главном конфиге][4].

Затем необходимо перезапустить `Nginx`. Для этого зайдите в терминал контейнера и перезапустите веб-сервер:

```shell
nginx -t
nginx -s reload
```

Проверьте, что ваше приложение отвечает по 443 порту.

## Настройка автообновления сертификата

После всех действий, когда приложение уже работает с выданным сертификатом, важно настроить его обновление.
Сертификат выдаётся на 3 месяца, поэтому настроим обновление раз в 2 месяца.

Для этого нужно в системе отредактировать cron задачи. Чтобы зайти в редактор, используйте:

```shell
crontab -e
```

Введите строку:

```shell
0 0 1 */2 * certbot renew --force-renewal && docker compose -f /home/danil/repo/skadi/docker-compose.yml restart web-server
```

Эта команда перевыпускает сертификат и перезапускает nginx для применения нового сертификата. Всё это будет происходить раз в 2 месяца.

## Firewall

После всех действий вы можете настроить файрвол.

[1]: ./examples/docker-compose.yml
[2]: ./examples/web-server/
[3]: ./examples/web-server/Dockerfile
[4]: ./examples/web-server/nginx.conf
[5]: ./examples/web-server/https_site.conf
