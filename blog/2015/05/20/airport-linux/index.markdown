---
tags: airplay, linux, shairport
title: Превращаем Linux компьютер в AirPlay приемник
---

Захотелось мне как-то играть музыку с мака на колонки без проводов. Раньше для этого я пользовался AirPort Express, када были воткнуты колонки. Но сия халява кончилась и теперь надо что-то изобретать.

Быстрый поиск по интернетам показал существование демона shairport, который предствляет собой AirPlay приемник для Linux. Проблема в том, что данный демон уже давно не поддерживается и автор прямо отказался от его поддержки. Но благодаря великой силе форков гитхаба был найден проект [shairport-sync](https://github.com/mikebrady/shairport-sync), который является форком shairport с допилами и поддерживается в настоящее время. То что надо!

Итак, краткий мануал по установке. Я взял старый asus eee 900, на котором стоит xubuntu 14.04LTS. По совместительству он работает у меня домашним сервером.

* Клонируем к себе репозиторий `git clone https://github.com/mikebrady/shairport-sync.git`
* Устанавливаем зависимости нужное для сборки `apt install build-essential autoconf checkinstall libtool libdaemon-dev libasound2-dev libpopt-dev`
* Устанавливаем Avahi `apt install avahi-daemon libavahi-client-dev`
* Устанавливаем зависимости для работы с SSL `apt install libssl-dev`. Демон умеет работать с OpenSSL или PolarSSL, но этот пакет нужен для обеих библиотек.
* Я использовал PolarSSL для сборки, но, в целом, никакой разницы нет. Устанавливаем PolarSSL `apt install libpolarssl-dev`
* Запускаем autoconf `autoreconf -i -f` после этого будет сгенерирован файл configure.
* `$ ./configure --with-alsa --with-avahi --with-ssl=polarssl`
* `make`
* `sudo checkinstall`. Checkinstall задаст несколько вопросов, на них можно ответить по умолчанию, единственное что надо задать поле `version` в виде числа. Т.к. по умолчанию туда попадает значение `sync` и checkinstall падает с ошибкой создания пакета.

Почему вместо `make install` лучше использовать `checkinstall`? Эта утилита перехватит работу make, отследит куда ставятся все файлы приложения и создаст deb пакет для безопасной установки и удаления приложения.

Конфигурируется демон через редактирования файла `/etc/init.d/shairport-sync`, я там поменял только название сервиса. Обычно больше ничего не требуется для работы.

### Возможные проблемы

При загрузке компьютера может появляться сообщение:

	Avahi detected that your currently configured local DNS server serves a domain .local. This is inherently incompatible with Avahi and thus Avahi disabled itself. If you want to use Avahi in this network, please contact your administrator and convince him to use a different DNS domain, since .local should be used exclusively for Zeroconf technology.

Работа Avahi будет останавливаться, соответственно и shairport не запустится. Обычно эта проблема связана с тем, что в DNS сервере есть записи для зоны local.

Проверяется это командой: `host -t SOA local.` (обратите внимание на точку в конце!). В нормальной ситуации вывод должен быть около такого: 

	bash-3.2$ host -t SOA local.
	Host local. not found: 3(NXDOMAIN)

Если же в выводе написано: `local has SOA record XXX`, то ваш DNS транслирует зону local. Обычно это делают провайдеры для работы сервисов типа retracker.local. Если вы этим не пользуетесь - можно просто указать днс Yandex (77.88.8.8/77.88.8.1) или Google(8.8.8.8/8.8.4.4).

После этих настроек и рестарта служб - все должно работать, а на устройствах Apple появится ваше устройство для вывода звука.
