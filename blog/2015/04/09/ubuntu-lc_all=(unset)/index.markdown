---
tags: 'Ubuntu, LC_ALL'
title: Ubuntu LC_ALL = (unset)
---

При установке Ubuntu (в основном на VDS) периодический вылезает ошибка:

    LANGUAGE = (unset)
    LC_ALL = (unset)

Для исправления делаем:

    # echo LC_ALL="ru_RU.UTF-8" >> /etc/environment
    # echo LC_CTYPE="ru_RU.UTF-8" >> /etc/environment
    # locale-gen ru_RU.UTF-8

И перелегониваемся.
