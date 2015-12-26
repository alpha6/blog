---
tags: perl, state, trap
title: State переменные в Perl
---

В Perl существует особый тип переменных под названием state.

В [доке](http://perldoc.perl.org/functions/state.html) про них написано:

> state declares a lexically scoped variable, just like my. However, those variables will never be reinitialized ...

На первый взгляд это дает нам возможность очень просто реализовывать счетчики и иже с ними:

    sub count {
      state $count = 0;
      $count++;
    }

Однако, есть нюанс - фраза *will never be reinitialized* означает что переменная действительно никогда не будет переинициализирована пока существует родительский скрипт. И это дает нам вот такую замечательную граблю на которую можно ненароком наступить:

Объявляем пакет:

    package MyTestState;

    use strict;
    use feature 'state';

    sub new {
      bless {}, shift;
    }

    sub count {
      state $count = 0;
      $count++;
    }

    1;

И саму программу:

    use lib 'lib';
    use v5.18;
    use MyTestState;

    my $mystate =  MyTestState->new();

    for (0..10) {
      my $counter = $mystate->count();
      say "Counter [$counter]";
    }

Вывод ожидаем:

    Counter [0]
    Counter [1]
    Counter [2]
    ...
    Counter [10]

А теперь добавляем такой код:

    undef $mystate;

    say "next object!=======";

    my $mystate1 = MyTestState->new();
    for (0..10) {
      my $counter = $mystate1->count();
      say "Counter [$counter]";

    }

Здесь мы удаляем старый объект со счетчиком и создаем новый. Логично предположить что счетчик пойдет заново, но на самом деле нет. Не смотря на то что мы удалили старый объект и создали новый, переменная со счетчиком никуда не делась и не была переиницализирована! И при запуске программы мы увидим:

    Counter [0]
    Counter [1]
    Counter [2]
    ...
    Counter [10]
    next object!
    Counter [11]
    Counter [12]
    Counter [13]
    ...
    Counter [20]
    Counter [21]

Так что слово ```newer``` в документации действительно значит "никогда пока жив инстанс интерпретатора запустивший скрипт".
