---
tags: perl, graylog
title: Используем Graylog с Perl
---

Устанавливаем Graylog. Для опытов я использовал преднастроенную VM для VirtualBox http://docs.graylog.org/en/1.0/pages/installation.html#virtual-machine-appliances

! На момент написания статьи в VM идет версия Graylog-web с багом - при создании dashboard на нее нельзя добавить виджет т.к. JS-скрипт ответственный за разблокировку dashboard падает с ошибкой. Что-бы что-то сделать с дэшбордом на него надо добавить любой виджет с любой страницы. Это делается по клику на иконку возле названия виджета и выбором нужного дэшборда.

Первым делом надо создать Input для логов.
Мы будем использовать GELF формат через Log::Log4perl.
Идем System->Inputs в выпадающем списке выбираем GELF UDP и жмем Launch Input.
В открывшемся окне выбираем ноды на которых будет работать этот инпут, описание, адрес на котором он будет слушать.
Жмем launch, убеждаемся что он стартанул и на этом пока все работы с Graylog закончены.

Теперь устанавливаем пакет [Log::Log4perl::Layout::GELF](https://metacpan.org/pod/Log::Log4perl::Layout::GELF).

Создаем файл конфигурации логгера:

	log4perl.logger.graylog                 	= INFO, Screen, Graylog
 
	log4perl.appender.Screen         		= Log::Log4perl::Appender::Screen
	log4perl.appender.Screen.stderr  		= 0
	log4perl.appender.Screen.layout 		= Log::Log4perl::Layout::PatternLayout
	log4perl.appender.Screen.layout.ConversionPattern = [%d] [%p] %m%n
	log4perl.appender.Screen.utf8     		= 1

	log4perl.appender.Graylog          = Log::Log4perl::Appender::Socket
	log4perl.appender.Graylog.PeerAddr = graylog.host
	log4perl.appender.Graylog.PeerPort = 12201
	log4perl.appender.Graylog.Proto    = udp
	log4perl.appender.Graylog.layout   = GELF


Создаем тестовый скрипт:

	#!/usr/bin/env perl

	use utf8;
	use strict;
	use Log::Log4perl;

	#Загружаем конфигурацию 
	Log::Log4perl::init('logger.conf');

	#Получаем логгер
	my $logger = Log::Log4perl->get_logger('graylog');

	$logger->info('hello graylog');
	$logger->info('Тестовое сообщение UTF8');


В теории, этого достаточно для того что бы нужные логи вашего приложения начали идти в Graylog. Но, как обычно, есть нюанс - этот модуль совершенно не представляет что есть более чем однобайтные кодировки и при попытке что-то записать в лог что-то с utf8 мы получим ошибку `Wide character in IO::Compress::Gzip::write` и в Graylog сообщение не придет.

Для обычных аппендеров, например Screen, эта проблема решается просто - дописываем в конфигурацию флаг включения utf8:

	log4perl.appender.FileAppndr.utf8     = 1

Но в данном случае проблема на уровне layout и этот модуль не обрабатывает такую ситуацию.

Для себя я эту проблему решил просто - сделал форк модуля с названием [GELFUtf](https://github.com/alpha6/Log-Log4perl-Layout-GELFUtf) и использую его.

Таким образом в конфиге вместо `log4perl.appender.Graylog.layout   = GELF` пишу `log4perl.appender.Graylog.layout   = GELFUtf`.

Подменить фунцию на лету у меня не получилось, скорее всего из-за хитрой архитектуры Log4perl. А лезть патчить в рантайм - овчинка выделки не стоит. Так что пока пользуюсь патченым модулем и коплю силы на создание полноценного патча для оригинального модуля.