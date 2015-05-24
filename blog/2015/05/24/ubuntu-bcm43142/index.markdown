---
tags: Ubuntu, note
title: ubuntu 15.04 и BCM43142
---

После апгрейда Ubuntu 14.04 на Kubuntu 15.04 выяснилось что Wi-Fi на моем ноутбуке работать не хочет от слова "совсем". И если в 14.04 после подключения к интернету он сам предлагал доустановить нужные драйвера - то в 15.04 магия не срабатывала и Wi-Fi не работал.

Симптомы:
* в NetworkManager нет даже пункта про беспроводные подключения.
* `# lshw -c Network` говорит что-то вроде:

 	*-network UNCLAIMED     
       description: Network controller
       product: BCM43142 802.11b/g/n
       vendor: Broadcom Corporation
       physical id: 0
       bus info: pci@0000:02:00.0
       version: 01
       width: 64 bits
       clock: 33MHz
       capabilities: bus_master cap_list
       configuration: latency=0
       resources: memory:90500000-90507fff

Как можно заметить - устройство карте не назначено. Значит надо доустановить пакет с прошивкой для карты. С 14.04 эта карта поддерживается в официальных драйверах, так что просто устанавливаем пакет руками:

	sudo apt-get install bcmwl-kernel-source

Перезагружемся и видим в NetworkManager раздел с беспроводными сетями.