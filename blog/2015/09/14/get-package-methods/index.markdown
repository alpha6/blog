---
tags: perl, note, package
title: Получение методов пакета
---

Для получения методов пакета Foo::Bar делаем:

    print Dumper(\%Foo::Bar::);

Для проверки существования метода:

    if (Foo::Bar::.$method_name) {
        #some stuff
    }

Для получения методов текущего пакета:

    print Dumper(\%main::)

Но если подключены дополнительные библиотеки - в выводе будут методы всех подключенных библиотек.

[Подробнее в документации](http://perldoc.perl.org/perlmod.html#Symbol-Tables)
