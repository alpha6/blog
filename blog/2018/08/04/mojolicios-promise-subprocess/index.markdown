---
tags: ~
title: Mojolicios+Promise+Subprocess
---

Иногда в веб-приложениях возникает необходимость сделать какое-то длительное действие, которое может привести к остановке эвент-лупа и, соответственно, отказу в обслуживании на время обработки запросов.

Для решения этой проблемы в приложениях на Mojolicious можно использовать комбинацию Promise и Subprocess в виде модулей [Mojo::Promise](https://mojolicious.org/perldoc/Mojo/Promise) и [Mojo::IOLoop](https://mojolicious.org/perldoc/Mojo/IOLoop/Subprocess).

Важный нюанс, если ваше длительное действие инзначально асинхронное и не нагружает процессор, то смысла в использовании subprocess нет, т.к. subprocess спавнится через fork, а это достаточно дорогое удовольствие. Так же у subprocess нет встроенных методов для ограничения нагрузки. Так что если CPU-Bound задачи вам надо решать часто, в больших количествах и, желательно, не устраивая DOS-атаку свой сервер, то лучше использовать очереди и воркеров. Хотя, конечно никто не мешает накидать очередь на промисах и вот это всё.

Итак, для примера создадим простейшее приложение на Mojo::Lite которое будет принимать картинку от пользователя и делать из нее набор картинок в разных разрешениях. 

```
#!/usr/bin/env perl

use strict;
use warnings;
use utf8;
use feature ':5.10';

use Mojolicious::Lite;

use Imager;
use File::Spec;

my $log = Mojo::Log->new;

my @scales     = qw/640 800 1024 2048/;
my $static_dir = app->static->paths->[0];

my %scales_paths = map { $_ => File::Spec->catfile( $static_dir, $_ ) } @scales;

# Upload form in DATA section
get '/' => 'form';

# Multipart upload handler
post '/upload' => sub {
    my $c = shift;

    # Process uploaded file
    return $c->redirect_to('form') unless my $image = $c->param('image');
    my $size = $image->size;
    my $name = $image->filename;

    mkdir $static_dir unless -e $static_dir;

    my $image_path = File::Spec->catfile( $static_dir, $name );

    $image->move_to($image_path);

    my $imager = Imager->new();
    $imager->read( file => $image_path ) or die $imager->errstr;

    for my $scale (@scales) {

        mkdir $scales_paths{$scale}
          unless -e $scales_paths{$scale};    #check that folder is exists

        my $scaled = $imager->scale( xpixels => $scale );

        $scaled->write(
            file => File::Spec->catfile( $scales_paths{$scale}, $name ) )
          or die $scaled->errstr;

    }

    $c->render( text => "Thanks for uploading $size byte file $name." );
};

app->start;
__DATA__

@@ form.html.ep
<!DOCTYPE html>
<html>
  <head><title>Upload</title></head>
  <body>
    %%= form_for upload => (enctype => 'multipart/form-data') => begin
      %%= file_field 'image'
      %%= submit_button 'Upload'
    %% end
  </body>
</html>
```
Основной код приложения честно сперт из [официального гайда](https://mojolicious.org/perldoc/Mojolicious/Guides/Tutorial#File-uploads). С добавлением кода обработки загруженных картинок. Конечно можно было ограничится банальным `sleep 5`, но, на мой взгляд, это скучно. Хоть и дает в разы меньше кода в примере.


Если запустить этот пример и загрузить картинку, то можно заметить что между нажатием кнопки Upload и получением результата проходит достаточно существенное время (на моем i5 около двух секунд для картинки в 12Mpix). Все это время приложение не отвечает на запросы снаружи, что может быть довольно неприятно. 

Первое и самое простое что можно сделать, если нам нет необходимости возвращать результат работы прямо сейчас, это просто обернуть ресурсоемкий код в subprocess.

Изменим метод для аплоада картинок чтобы он выглядел следующим образом:

```
post '/upload' => sub {
    my $c = shift;

    # Process uploaded file
    return $c->redirect_to('form') unless my $image = $c->param('image');
    my $size = $image->size;
    my $name = $image->filename;

    mkdir $static_dir unless -e $static_dir;

    my $image_path = File::Spec->catfile( $static_dir, $name );

    $image->move_to($image_path);

    save_image($name, $image_path);  

    $c->render( text => "Thanks for uploading $size byte file $name." );
};

Mojo::IOLoop->start;
app->start;

sub save_image {
  my $image_name = shift;
  my $image_path = shift;


  my $imager = Imager->new();
    $imager->read( file => $image_path ) or die $imager->errstr;

    Mojo::IOLoop->subprocess(
        sub {
            my $subprocess = shift;

            for my $scale (@scales) {

                mkdir $scales_paths{$scale}
                  unless -e $scales_paths{$scale};  #check that folder is exists

                my $scaled = $imager->scale( xpixels => $scale );

                $scaled->write(
                    file => File::Spec->catfile( $scales_paths{$scale}, $image_name )
                ) or die $scaled->errstr;

            }
            $log->debug("done");
            return 1;
        },
        sub {
            my ( $subprocess, $err, @results ) = @_;
            $log->error("Subprocess error: $err") and return if $err;
        }
    );
}
```
 
Здесь мы вынесли код сохранения картинки в отдельную подпрограмму, а заодно обернули в subprocess. Кстати, сохранять вотчер сабпроцесса не требуется. При создании он сам прописывается в IOLoop и живет там.


Теперь можно заметить, что ответ в браузер выдается практически сразу, а строчка `done` в логе появляется позже ответа `[debug] 200 OK (0.201017s, 4.975/s)`. В списке процессов тоже можно будет наблюдать появление чайлда у нашего процесса приложения. Так работает subprocess.

Ок, с простым запуском разобрались, теперь посмотрим что делать если мы хотим выдать клиенту ответ по результату работы, но при этом хотим обслуживать других клиентов. Здесь нам поможет Promise в виде модуля `Mojo::Promise`, который реализует спецификацию [Promise/A+](https://promisesaplus.com/).


Приведем код к виду:

```
post '/upload' => sub {
    my $c = shift;

    # Process uploaded file
    return $c->redirect_to('form') unless my $image = $c->param('image');
    my $size = $image->size;
    my $name = $image->filename;

    mkdir $static_dir unless -e $static_dir;

    my $image_path = File::Spec->catfile( $static_dir, $name );

    $image->move_to($image_path);

    my $promise = save_image( $name, $image_path );

    $promise->then(
        sub {
            $c->render( text => "Thanks for uploading $size byte file $name." );
        }
    )->wait;
};
```
```
sub save_image {
    my $image_name = shift;
    my $image_path = shift;

    my $imager = Imager->new();
    $imager->read( file => $image_path ) or die $imager->errstr;

    my $promise = Mojo::Promise->new;
    Mojo::IOLoop->subprocess(
        sub {
            my $subprocess = shift;

            for my $scale (@scales) {

                mkdir $scales_paths{$scale}
                  unless -e $scales_paths{$scale};  #check that folder is exists

                my $scaled = $imager->scale( xpixels => $scale );

                $scaled->write( file =>
                      File::Spec->catfile( $scales_paths{$scale}, $image_name )
                ) or die $scaled->errstr;

            }
            $log->debug("done");
            return 1;
        },
        sub {
            my ( $subprocess, $err, @results ) = @_;
            $promise->reject("Subprocess error: $err @results") if $err;
            $promise->resolve( 1, "done" );
        }
    );
    return $promise;
}
```

По сравнению с прошлым вариантом тут добавилось создание объекта Promise в save_image и данные клиенту мы возвращаем уже из Promise, а не напрямую из контроллера. Так же при возникновении ошибки в subprocess мы не возвращаем undef, а режектим promise. В результате работы этого кода мы спавним сабпроцесс, но ответ клиенту не выдаем пока он не отработает. При этом наше приложение продолжает обслуживать других клиентов. 

И опять я напомню что в таком варианте у нас нет контроля количества запущенных процессов и заДОСить свой сервер легче легкого!

Что же делать если нам надо гарантированно быстро положить сервер? :) Ведь в таком варианте мы создаем только один subprocess на запрос. Конечно же создавать несколько! 
Выглядеть это будет так:

```
post '/upload' => sub {
    my $c = shift;

    # Process uploaded file
    return $c->redirect_to('form') unless my $image = $c->param('image');
    my $size = $image->size;
    my $name = $image->filename;

    mkdir $static_dir unless -e $static_dir;

    my $image_path = File::Spec->catfile( $static_dir, $name );

    $image->move_to($image_path);

    my $promise_first = save_image( $name, $image_path, @scales[0..$#scales/2] );
    my $promise_second = save_image( $name, $image_path, @scales[$#scales/2..$#scales]);

    Mojo::Promise->all($promise_first, $promise_second)->then(
        sub {
            $c->render( text => "Thanks for uploading $size byte file $name." );
        }
    )->wait;
};
```
```
sub save_image {
    my $image_name = shift;
    my $image_path = shift;
    my @scales = @_;

    ...
}

```

Здесь мы делаем два вызова метода `save_image` с разным набором размеров для ресайза, каждый из которыйх возвращает свой promise.
Затем дожидаемся когда они оба отработают в `Mojo::Promise->all($promise_first, $promise_second)` и после этого выдаем ответ клиенту. 

В примерах выше мы не ждали никаких результатов от subprocess, в реальной жизни это относительно редкое явление. В данном случае все вообще элементарно.

```
    Mojo::IOLoop->subprocess(
        sub {
            my $subprocess = shift;

            ...

            return $work_results;
        },
        sub {
            my ( $subprocess, $err, @results ) = @_;
            $promise->reject("Subprocess error: $err @results") if $err;
            $promise->resolve( @results );
        }
    );
```

И в коде обработки promise:

```
    Mojo::Promise->all(@promises_list)->then(
        sub {
          for my $prom( @_) {
            $log->debug("promise result: $prom->[1]") 
          }
        }
    )->wait;
```
На вход сабы приходит массив результатов работ промисов.
Соответственно, что вернули в `$promise->resolve( @results );`, то и будет лежать. Ну помноженное на кол-во промисов.

Еще один момент - обработка ошибок. В целом тут все просто. Если промис был режектнут или в нем возникло исключение, то promise попадает в блок `catch` где происходит обработка ошибки.

```
Mojo::Promise->all(@promises_list)->then(
        sub {
          for my $prom( @_) {
            $log->debug("promise result: $prom->[1]") 
          }
        }
    )->catch(
        sub {
            my $err = shift;
            $log->error("promise error: $err);
        }
    )->wait;
```

Важный момент, если мы попадаем в catch, то в then мы уже не попадем. То есть, например, запустить десяток запросов к разным серверам, а потом показать только успешные не получится. Для таких кейсов придется использовать другие механизмы.

Еще нюанс: в случае если блок `catch` отсутствует и какой-то из промисов будет режектнут, то `all()->then` не вызовется никогда! Так что надо не забывать его добавлять.

В общем и целом это достаточно полезная комбинация когда нам надо изредка(!) выполнять тяжелые запросы блокирующие приложение. Если такие запросы будут не редкие, то лучше использовать очереди. Например, тот же Minion, если не хочется заморачиваться со всякими RabbitMQ и прочим кровавым энтерпрайзом.

Ну и напоследок финальный код приложения:

```
#!/usr/bin/env perl

use strict;
use warnings;
use utf8;
use feature ':5.10';

use Mojolicious::Lite;

use Imager;
use File::Spec;

my @scales     = qw/640 800 1024 2048/;
my $static_dir = app->static->paths->[0];

my %scales_paths = map { $_ => File::Spec->catfile( $static_dir, $_ ) } @scales;

my $log = Mojo::Log->new;

# Upload form in DATA section
get '/' => 'form';

# Multipart upload handler
post '/upload' => sub {
    my $c = shift;

    return $c->redirect_to('form') unless my $image = $c->param('image');
    my $size = $image->size;
    my $name = $image->filename;

    mkdir $static_dir unless -e $static_dir;

    my $image_path = File::Spec->catfile( $static_dir, $name );

    $image->move_to($image_path);

    my $promise_first = save_image( $name, $image_path, @scales[0..$#scales/2] );
    my $promise_second = save_image( $name, $image_path, @scales[$#scales/2..$#scales]);

    Mojo::Promise->all($promise_first, $promise_second)->then(
        sub {
            $c->render( text => "Thanks for uploading $size byte file $name." );
        }
    )->catch(
        sub {
            my $err = shift;
            $c->render( text => "One of promises died :( $err" );
        }
    )->wait;
};

Mojo::IOLoop->start;
app->start;

sub save_image {
    my $image_name = shift;
    my $image_path = shift;
    my @scales = @_;

    my $imager = Imager->new();
    $imager->read( file => $image_path ) or die $imager->errstr;

    my $promise = Mojo::Promise->new;
    Mojo::IOLoop->subprocess(
        sub {
            my $subprocess = shift;

            for my $scale (@scales) {

                mkdir $scales_paths{$scale}
                  unless -e $scales_paths{$scale};  #check that folder is exists

                my $scaled = $imager->scale( xpixels => $scale );

                $scaled->write( file =>
                      File::Spec->catfile( $scales_paths{$scale}, $image_name )
                ) or die $scaled->errstr;

            }
            $log->debug("done @scales");
            return 1;
        },
        sub {
            my ( $subprocess, $err, @results ) = @_;
            $promise->reject("Subprocess error: $err @results") if $err;
            $promise->resolve( 1, "done" );
        }
    );
    return $promise;
}
__DATA__

@@ form.html.ep
<!DOCTYPE html>
<html>
  <head><title>Upload</title></head>
  <body>
    %%= form_for upload => (enctype => 'multipart/form-data') => begin
      %%= file_field 'image'
      %%= submit_button 'Upload'
    %% end
  </body>
</html>
```

