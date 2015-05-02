---
tags: perl, graylog
title: Используем Graylog с Perl
---

Устанавливаем Graylog. Для опытов я использовал преднастроенную VM для VirtualBox http://docs.graylog.org/en/1.0/pages/installation.html#virtual-machine-appliances

! На момент написания статьи в VM идет версия Graylog-web с багом - при создании dashboard на нее нельзя добавить виджет т.к. JS-скрипт ответственный за разблокировку dashboard падает с ошибкой.

Первым делом надо создать Input для логов.
Мы будем использовать GELF формат через Log::Log4perl.
Идем System->Inputs в выпадающем списке выбираем GELF UDP и жмем Launch Input.
В открывшемся окне выбираем ноды на которых будет работать этот инпут, описание, адрес на котором он будет слушать.
Жмем launch, убеждаемся что он стартанул и на этом пока все работы с Graylog закончены.

