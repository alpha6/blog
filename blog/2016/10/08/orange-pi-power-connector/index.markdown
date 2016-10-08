---
tags: orangepi, diy, power
title: Orange Pi power connector
---
Orange Pi can't work with USB power. It needs power from power connector or GPIO.

If you need a fast and dirty way, use the GPIO pins. You may use standard smartphone charger with USB connector and 1A current.
If you want to use external HDD with Orange Pi, you need power supply with 2+ A current.
After you have found a charger, just connect +5V from USB to GPIO pin 2 (or pin 4, they are similar) and USB ground with pin 6.

![GPIO pinout diagram](http://cs5-3.4pda.to/8498940.png)

The first pin is marked with white triangle near the GPIO connector.

The right way is to use the power plug. But Orange Pi uses power connector which is not common.
You need the connector with 4.0x1.7 mm diameter and 8mm long (or longer). In the shop where I bought the connector it was being sold as connector for Compaq notebooks.
You need to solder +5V from the USB to the central contact of the connector and USB ground to the external contact.
