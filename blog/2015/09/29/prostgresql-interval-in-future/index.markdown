---
tags: postgresql, data, timestamp
title: postgresql interval in future
---
Чтобы получить значение времени в заданном интервале от текущего момента можно использовать следующий синтаксис:

    SELECT NOW() + '5 minutes'::interval;
    SELECT NOW() + '5 days'::interval;

Работает и в плюс и в минус.
