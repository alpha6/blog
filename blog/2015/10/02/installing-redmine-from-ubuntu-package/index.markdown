---
tags: redmine, postgresql, nginx, ubuntu
title: Installing Redmine from Ubuntu package
---

Устанавливаем Redmine из пакетов в Ubuntu. Версия в пакетах не самая свежая, зато никаких заморочек с настройкой руби и интеграцией с БД.

В качестве БД будем использовать Postgresql, на фронтенде Nginx, все совершенно стандартно.

Устанавливаем все нужные пакеты:

    sudo apt install redmine redmine-psql postgresql nginx thin

В вопросах инсталятора `redmine` выбираем бэкэнд `psql`, в следующем диалоге на запрос автоматической конфигурации БД говорим `yes`, оставляем поле пароля пустым. Пароль будет сгенерирован автоматически и данные зальются в схему `redmine_default`.

`Redmine` установится в `/usr/share/redmine`, его конфигурация будет лежать в `/etc/redmine` но сейчас там нет ничего интересного для нас.

Проверим что все работает корректно. Идем в `/usr/share/redmine` и запускаем `Redmine` с помощью `webrick`

     ruby script/rails server webrick -e production

Демон запустится на `0.0.0.0:3000` заходим, проверяем что все работает, гасим демона.

Теперь настроим `thin` для запуска `Redmine`. Создадим конфиг файл:

    sudo thin config --config /etc/thin1.9.1/redmine.yml --chdir YOUR_REDMINE_DIRECTORY \
    --environment production --address 0.0.0.0 --port 3030 \
    --daemonize --log /var/log/thin/redmine.log --pid /var/run/thin/redmine.pid \
    --user www-data --group www-data --servers 1 --prefix YOUR_PREFIX

`YOUR_REDMINE_DIRECTORY` - путь к директории куда установлен `Redmine`, в нашем случае `/usr/share/redmine`.
`YOUR_PREFIX` - префикс в адресе по которому будет доступен `Redmine`, для установки в `http://your_redmine.domain/` пишем `/`. Если хотим чтобы доступ был через `http://your_redmine.domain/redmine` пишем `redmine`.

Команда для установки в корень по умолчанию будет выглядеть как-то так:

    sudo thin config --config /etc/thin1.9.1/redmine.yml --chdir /usr/share/redmine --environment production --address 0.0.0.0 --port 3030 --daemonize --log /var/log/thin/redmine.log --pid /var/run/thin/redmine.pid --user www-data --group www-data --servers 1 --prefix /

Теперь настроим `Nginx`.

Создадим файл `/etc/nginx/sites-available/redmine` со следующим содержимым:

    upstream redmine_thin_servers {
    	server 0.0.0.0:3030;
    }

    server {

      listen   80; ## listen for ipv4

      server_name your_redmine.domain;
      server_name_in_redirect off;

      proxy_set_header        Host $http_host;
      proxy_set_header        X-Real-IP $remote_addr;
      proxy_set_header        X-Forwarded-For $proxy_add_x_forwarded_for;
      proxy_set_header        X-Forwarded-Proto $scheme;
      proxy_redirect off;

      location / {
        root   /usr/share/redmine/public;

        error_page 404  404.html;
        error_page 500 502 503 504  500.html;

        try_files $uri/index.html $uri.html $uri @redmine_thin_servers;
      }

      location @redmine_thin_servers {
        proxy_pass http://redmine_thin_servers;
      }
    }

Создадим на него ссылку в `/etc/nginx/sites-enabled`

    cd /etc/nginx/sites-enabled
    sudo ln -s ../sites-available/redmine

Запустим `thin` и `nginx`

    sudo service thin start
    sudo service nginx start

После этого `redmine` должен быть доступен на 80м порту вашего домена. Если там 502 или еще чего - значит конфигурация отличается и надо смотреть какую ошибку пишут в логах `thin` и `redmine`.

Теперь закроем все это дело firewall. Так как никаких творческих решений нам не нужно, а нужно чтобы работало - то будем использовать `ufw`

Оставим открытыми только ssh и http(80) порты

  sudo apt install ufw
  ufw allow ssh
  ufw allow http
  ufw enable

Это установит ufw, добавит ssh и http в список разрешенных портов, заблокирует все остальные и добавит правил firewall в автозагрузку при старте системы.

Enjoy.
