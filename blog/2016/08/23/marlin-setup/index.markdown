---
tags: prusai3, 3dprinting, ramps, marlin
title: Настраиваем прошивку Marlin для Arduino + RAMPS 1.4
---

Выбираем нашу плату: RAMPS 1.4 с одним хотэндом

    #ifndef MOTHERBOARD
      #define MOTHERBOARD BOARD_RAMPS_14_EFB
    #endif

Выставляем температурный сенсор.

    #define TEMP_SENSOR_0 1

Выставляем максимальную температуру хотэнда и стола

    #define HEATER_0_MAXTEMP 250
    #define BED_MAXTEMP 130

Включаем инвертирование концевиков

    // Mechanical endstop with COM to ground and NC to Signal uses "false" here (most common setup).
    #define X_MIN_ENDSTOP_INVERTING true // set to true to invert the logic of the endstop.
    #define Y_MIN_ENDSTOP_INVERTING true // set to true to invert the logic of the endstop.
    #define Z_MIN_ENDSTOP_INVERTING true // set to true to invert the logic of the endstop.

Проверяем направление вращения моторов

    // Invert the stepper direction. Change (or reverse the motor connector) if an axis goes the wrong way.
    #define INVERT_X_DIR true
    #define INVERT_Y_DIR true
    #define INVERT_Z_DIR false

и экструдера

    #define INVERT_E0_DIR true

Выставляем размеры рабочей зоны

    #define X_MIN_POS 0
    #define Y_MIN_POS 0
    #define Z_MIN_POS 0
    #define X_MAX_POS 250
    #define Y_MAX_POS 230
    #define Z_MAX_POS 140

Устанавливаем координаты начала стола

    #define MANUAL_X_HOME_POS -30
    #define MANUAL_Y_HOME_POS -20

Выставляем шаги для моторов

    #define DEFAULT_AXIS_STEPS_PER_UNIT   {100,100,1600,95}
