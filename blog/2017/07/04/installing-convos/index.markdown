---
tags: perl, convos, irc
title: Installing Convos
---

Современные технологии это всегда хорошо, но и нестареющая классика на то и классика чтобы про нее не забывать. В данном случае я говорю об IRC.
IRC прекрасная технология, вот только мобильные клиенты для него отличаются кривостью и нестабильностью работы. А уж зайти в IRC из корпоративных сетей может быть еще тем квестом.
Плюс особенностью IRC является тот факт, что никакой истории сообщений там не хранится и все что было в ваше отсутствие на канале пройдет мимо вас.
Частично решить первую проблему и полностью вторую и третью предназначен [Convos](https://convos.by).

Это веб интерфейс для IRC. Умеет поддерживать много соединений. Хранит историю и дает приятный веб-интерфейс неплохо работающий на смартфонах.

Будем ставить его из гита на свой сервер. Для таких целей я использую OpenVZ сервера от [Time4vps](https://billing.time4vps.eu/?affid=1845). Серваки в Европе, стоят дешево, работают достаточно шустро и стабильно. Для наших целей достаточно самого дешевого.

Считаем что сервер у нас есть и даже с доменным именем.

Настроим SSL, нам он нужен только для того что-бы наш трафик не читал кто попало, так что хватит сертификата от [LetsEncript](https://letsencrypt.org). Тем более что получение и обновление сертификата нынче автоматизировано во всех более-менее современных дистрибутивах.

# Настраиваем nginx и SSL

Устанавливаем certbot:

    apt-get install certbot

Зарегистрируемся в сервисе (надо сделать один раз):

    certbot register --email me@example.com

Настроим nginx для поддержки автоматического обновления сертификатов:

    mkdir -p /var/www/html/.well-known/acme-challenge
    echo Success > /var/www/html/.well-known/acme-challenge/example.html

Создадим инклюд для нашего локейшена для обновления сертификатов:

    vim /etc/nginx/acme

    location /.well-known {
        root /var/www/html;
    }

Заворачиваем все запросы (кроме обновления сертификатов) на https:

    vim /etc/nginx/sites-aviable/default

    server {
        listen 80 default_server;

        include acme;

        location / {
            return 301 https://$host$request_uri;
        }
    }

Перезагружаем nginx и проверяем что локейшен доступен:

    service nginx reload

    curl -L http://irc.example.com/.well-known/acme-challenge/example.html
    Success

Удаляем файл который мы создали для теста:

    rm http://irc.example.com/.well-known/acme-challenge/example.html

В принципе он ни на что не влияет, но с ним certbot будет ругаться что не удалось удалить все лишнее после обновления сертификатов.

Создаем конфиг файл для certbot, чтобы не задавал лишних вопросов и работал автоматом:

    vim /etc/letsencrypt/cli.ini

    authenticator = webroot
    webroot-path = /var/www/html
    post-hook = service nginx reload
    text = True
    agree-tos = True
    email = imdefined@yandex.ru


Проверяем что все работает как надо:

    certbot certonly --dry-run -d irc.example.com

    Saving debug log to /var/log/letsencrypt/letsencrypt.log
    Starting new HTTPS connection (1): acme-staging.api.letsencrypt.org
    Cert not due for renewal, but simulating renewal for dry run
    Renewing an existing certificate
    Performing the following challenges:
    http-01 challenge for irc.example.com
    Using the webroot path /var/www/html for all unmatched domains.
    Waiting for verification...
    Cleaning up challenges
    Unable to clean up challenge directory /var/www/html/.well-known/acme-challenge
    Generating key (2048 bits): /etc/letsencrypt/keys/0004_key-certbot.pem
    Creating CSR: /etc/letsencrypt/csr/0004_csr-certbot.pem
    Running post-hook command: service nginx reload

    IMPORTANT NOTES:
    - The dry run was successful.
    - If you lose your account credentials, you can recover through
    e-mails sent to imdefined@yandex.ru.
    - Your account credentials have been saved in your Certbot
    configuration directory at /etc/letsencrypt. You should make a
    secure backup of this folder now. This configuration directory will
    also contain certificates and private keys obtained by Certbot so
    making regular backups of this folder is ideal.

Все ок, получаем полноценный сертификат:

    certbot certonly -d irc.example.com

    Saving debug log to /var/log/letsencrypt/letsencrypt.log
    Starting new HTTPS connection (1): acme-v01.api.letsencrypt.org
    Obtaining a new certificate
    Performing the following challenges:
    http-01 challenge for irc.example.com
    Using the webroot path /var/www/html for all unmatched domains.
    Waiting for verification...
    Cleaning up challenges
    Generating key (2048 bits): /etc/letsencrypt/keys/0004_key-certbot.pem
    Creating CSR: /etc/letsencrypt/csr/0004_csr-certbot.pem

    IMPORTANT NOTES:
        - Congratulations! Your certificate and chain have been saved at
        /etc/letsencrypt/live/irc.example.com/fullchain.pem. Your cert will
        expire on 2017-10-09. To obtain a new or tweaked version of this
        certificate in the future, simply run certbot again. To
        non-interactively renew *all* of your certificates, run "certbot
        renew"

Теперь создадим конфиг nginx для сервиса:


    vim /etc/nginx/sites-available/irc.example.com

    server {
        listen 443 ssl;

        ssl on;
        ssl_stapling on;

        server_name irc.example.com;

        ssl_certificate     /etc/letsencrypt/live/irc.example.com/fullchain.pem;
        ssl_certificate_key /etc/letsencrypt/live/irc.example.com/privkey.pem;
        
        ssl_ciphers 'ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-AES256-GCM-SHA384:DHE-RSA-AES128-GCM-SHA256:DHE-DSS-AES128-GCM-SHA256:kEDH+AESGCM:ECDHE-RSA-AES128-SHA256:ECDHE-ECDSA-AES128-SHA256:ECDHE-RSA-AES128-SHA:ECDHE-ECDSA-AES128-SHA:ECDHE-RSA-AES256-SHA384:ECDHE-ECDSA-AES256-SHA384:ECDHE-RSA-AES256-SHA:ECDHE-ECDSA-AES256-SHA:DHE-RSA-AES128-SHA256:DHE-RSA-AES128-SHA:DHE-DSS-AES128-SHA256:DHE-RSA-AES256-SHA256:DHE-DSS-AES256-SHA:DHE-RSA-AES256-SHA:AES128-GCM-SHA256:AES256-GCM-SHA384:AES128-SHA256:AES256-SHA256:AES128-SHA:AES256-SHA:AES:CAMELLIA:DES-CBC3-SHA:!aNULL:!eNULL:!EXPORT:!DES:!RC4:!MD5:!PSK:!aECDH:!EDH-DSS-DES-CBC3-SHA:!EDH-RSA-DES-CBC3-SHA:!KRB5-DES-CBC3-SHA';
        ssl_prefer_server_ciphers on;

        ssl_dhparam /etc/nginx/conf.d/dhparams.pem;

        error_log  /var/log/nginx/irc.example.com.error.log error;


        include acme;

        location / {
            proxy_pass      http://127.0.0.1:3001;
            access_log     /var/log/nginx/irc.example.com.log combined;
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection "upgrade";

            # Enable Convos to construct correct URLs by passing on custom
            # headers. X-Request-Base is only required if "location" above
            # is not "/".
            proxy_set_header Host $host;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;

        }

    }

Активируем этот сайт:

    ln -s /etc/nginx/sites-available/irc.example.com /etc/nginx/sites-enabled/

Сгенерим усиленные параметры шифрования для DHE:

    openssl dhparam -out /etc/ssl/private/dhparam.pem 2048

Добавим их в конфигурацию nginx:

    vim /etc/nginx/conf.d/ssl_parameters.conf

    ssl_dhparam /etc/ssl/private/dhparams.pem;

Перезапускаем nginx и наслаждаемся ошибкой 502 по тому адресу где должен быть сайт :)

    service nginx restart

# Настроим firewall

Создадим system.d сервис который будет поднимать firewall при старте сети:

    vim /etc/systemd/system/firewall.service

    [Unit]
    Description=Add Firewall Rules to iptables

    [Service]
    Type=oneshot
    ExecStart=/etc/firewall/enable.sh

    [Install]
    WantedBy=multi-user.target

Firewall я тут привожу самый простой, блокирует доступ ко всем портам кроме 22, 80, 443 и всех выпускает наружу:

    vim /etc/firewall/enable.sh

    #!/bin/sh

    # iptables script generated 2017-07-04
    # http://www.mista.nu/iptables

    IPT="/sbin/iptables"

    # Flush old rules, old custom tables
    $IPT --flush
    $IPT --delete-chain

    # Set default policies for all three default chains
    $IPT -P INPUT DROP
    $IPT -P FORWARD DROP
    $IPT -P OUTPUT ACCEPT

    # Enable free use of loopback interfaces
    $IPT -A INPUT -i lo -j ACCEPT
    $IPT -A OUTPUT -o lo -j ACCEPT

    # All TCP sessions should begin with SYN
    $IPT -A INPUT -p tcp ! --syn -m state --state NEW -s 0.0.0.0/0 -j DROP

    # Accept inbound TCP packets
    $IPT -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
    $IPT -A INPUT -p tcp --dport 22 -m state --state NEW -s 0.0.0.0/0 -j ACCEPT
    $IPT -A INPUT -p tcp --dport 80 -m state --state NEW -s 0.0.0.0/0 -j ACCEPT
    $IPT -A INPUT -p tcp --dport 443 -m state --state NEW -s 0.0.0.0/0 -j ACCEPT



# Поднимаем сервис Convos

Создадим пользователя convos. От его имени будет работать сервис.

Устанавливаем [perlbrew](https://perlbrew.pl) (Этот шаг можно пропустить если у вас относительно свежий дистрибутив, но я предпочитаю контролировать среду где у меня работает софт).
Perlbrew ставим от root в систему чтобы он был виден все пользователям:

    apt-get install perlbrew
    perlbrew init
    source /opt/perlbrew/etc/bashrc

    echo source /opt/perlbrew/etc/bashrc >> /etc/bash.bashrc

Устанавливаем perl 5.24.1 и переключаемся на него чтобы поставить пару модулей (на момент написания статьи Convos не ставился на 5.26 из-за ошибки в одном из сторонних модулей).

    perlbrew install perl-5.24.1
    perlbrew use perl-5.24.1
    cpan -i Carton

После установки perl и [Carton](https://metacpan.org/pod/Carton) можно настраивать все дальше.

Для начала создадим system.d сервис который будет поднимать наш сервис после старта nginx (потому что если nginx не поднялся все равно ничего работать не будет).

    vim /etc/systemd/system/convos.service

    [Unit]
    After=nginx.service
    Description=ConvosService

    [Service]
    ExecStart=/srv/convos/start.sh
    WorkingDirectory=/srv/convos
    User=convos
    Group=convos
    Restart=always

    [Install]
    WantedBy=default.target

Клонируем convos из гита в /srv/convos

    cd /srv/
    git clone https://github.com/Nordaaker/convos.git

Переключаемся на пользователя convos и ставим все зависимости через Carton:

    su convos
    cd /srv/convos 
    export PERLBREW_ROOT=/opt/perlbrew
    export PERLBREW_HOME=/srv/convos/.perlbrew_convos
    source ${PERLBREW_ROOT}/etc/bashrc

    carton install

После того как поставятся все зависимости создадим враппер для запуска сервиса Convos

    vim /srv/convos/start.sh 
    #!/bin/bash

    ## These 3 lines are mandatory.
    export PERLBREW_ROOT=/opt/perlbrew
    export PERLBREW_HOME=/srv/convos/.perlbrew_convos
    source ${PERLBREW_ROOT}/etc/bashrc

    ## Do stuff with 5.24.1
    perlbrew use 5.24.1
    export MOJO_REVERSE_PROXY=1
    #export CONVOS_DEBUG=1
    carton exec ./script/convos daemon --listen http://127.0.0.1:3001

Активируем сервис system.d и проверяем его статус: 

    systemctl enable convos
    systemctl -l status convos

Для начала лучше запустить враппер руками от пользователя convos и убедится что нет ошибок запуска (заодно посмотреть инвайт код для регистрации через веб).

Если все в порядке - то запускаем сервис через system.d и наслаждаемся прогрессивной работой в IRC. 

Если сервис таки не запустился, то посмотреть ошибки можно командой:

    journalctl -u convos -f


## Make the IRC great again!