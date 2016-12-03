---
tags: java, json, apache.http, confluence
title: Корректное получение JSON ответа с помощью Apache.HttpClient
---

У Apache HttpClient есть небольшая, но неприятная бага. Всплыла она при работе с REST API Confluence (вообще очень много баг всплывает при работе с API  продуктов Atlassian, но это все лирика).

В чем, собственно, проблема? Проблема в том, что Confluence при запросе у него данных с Accept: application/json (собственно все JSON REST API) не выставляет заголовок content-encoding для отдаваемых данных. В целом, его можно понять т.к. по стандарту JSON всегда должен быть UTF-8, но HttpClient не понимает и потому разбирает входящие данные как ISO.

Соответственно, дальше JSONObject при парсинге Entity превращает русский текст в совершенно непотребное.

Лечится это элементарно, просто при работае с Entity принудительно указываем ей что кодировка у нас UTF-8.

Вот так:

    JSONObject jsonObj = new JSONObject(EntityUtils.toString(response.getEntity, "UTF-8"));

После этого русский текст внутри JSON будет в нормальном UTF-8 и с ним можно спокойно работать.
