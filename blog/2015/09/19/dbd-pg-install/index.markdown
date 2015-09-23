---
tags: Ubuntu, Postgresql, Perl, DBD::Pg
title: DBD::Pg install
---
Markdown content goes here.

Если при установке `DBD::Pg` через `CPAN` у вас начинают спрашивать какие-то странные слова про номер версии и расположение директорий Postgresql - проверьте что у вас установлен пакет `postgresql-server-dev-X.X` (`postgresql-server-dev-all`).

После его установки проблема магическим (на самом деле нет) образом исчезает.