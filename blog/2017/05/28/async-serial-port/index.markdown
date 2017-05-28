---
tags: AnyEvent, perl, serial port, hardware
title: Асинхронная работа с COM-портом в Perl
---
Понадобилось мне тут поработать с 3d принтером из Perl. Принтер подключается к компьютеру через USB-COM переходник и прикидывается обычным COM портом со всеми вытекающими способами работы.

Для работы с ком-портом в Perl есть отличный модуль - [Device::SerialPort](https://metacpan.org/pod/Device::SerialPort). А для асинхронности используем классический AnyEvent. Ну и долго сказка сказывается, да быстро код пишется - пример кода:


    #!/usr/bin/env perl

    use v5.20;
    use strict;

    use AnyEvent;
    use AnyEvent::Handle;

    use Device::SerialPort;

    my $cv = AE::cv;

    # Базовые параметры подключения к порту
    my $device_port = '/dev/ttyUSB0';
    my $port_speed  = 115200;

    say "Connecting.. [$device_port] [$port_speed]";
    my $port = Device::SerialPort->new($device_port);
    $port->baudrate($port_speed); #Устанавливаем скорость соединения

    # Чуть более продвинутые настройки
    $port->handshake("none"); #Не используем handshake иначе подключение будет устанавливаться только в момент перезагрузки принтера

    # Режим коммуникации 8N1
    $port->databits(8);
    $port->parity("none");
    $port->stopbits(1);

    $port->stty_echo(0); # Выключаем эхо
    $port->error_msg('ON'); # Включаем выдачу ошибок от порта

    # Получаем чистый хэндлер порта и с ним создаем объект AE::Handle
    my $fh = $port->{'HANDLE'};

    my $handle;
    $handle = AnyEvent::Handle->new(
    fh       => $fh,
    on_error => sub {
        my ( $handle, $fatal, $message ) = @_;
        $handle->destroy;
        undef $handle;
        say STDERR "$fatal : $message\n";
    },
    on_read => sub {
        my $printer_handle = shift;
        $handle->push_read(
            line => sub {
                my ( $printer_handle, $line ) = @_;
                say sprintf( "Reply: [%s]", $line );
            }
        );
    }
    );

    # Отправляем команду принтеру
    $port->write("M105\n");

    $cv->recv;

После запуска (если все параметры указаны верно) получим такой вывод:

    Connecting.. [/dev/ttyUSB0] [115200]
    Reply: [ok T:26.6 /0.0 @:0]

Подключение к принтеру и передача данных в обе стороны прошли успешно!

**Важный нюанс!**

Хэндлер порта полученный тут ```my $fh = $port->{'HANDLE'};``` **однонаправленный на чтение**! Если попытаться туда что-то записать силами AE то получим ошибку. Писать надо напрямую в объект порта, что и происходит в предпоследней строке.
