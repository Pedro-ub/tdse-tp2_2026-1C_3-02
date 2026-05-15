# TP2 - Actividad 02 - Sensor Statechart

## Descripción
Implementación del modelo Sensor para 3 botones:

- BTN_B
- BTN_C
- BTN_D

Cada sensor implementa debounce mediante máquina de estados.

## Estados implementados
- ST_BTN_UP
- ST_BTN_FALLING
- ST_BTN_DOWN
- ST_BTN_RISING

## Eventos implementados
- EV_BTN_UP
- EV_BTN_DOWN

## Configuración de hardware
Botones configurados como:
- GPIO input
- Pull-up interno
- Active Low


## Observaciones
Se verificó funcionamiento correcto del debounce y generación de eventos hacia task_system.