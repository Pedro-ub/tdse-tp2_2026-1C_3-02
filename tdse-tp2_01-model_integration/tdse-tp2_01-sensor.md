# TP2 - Actividad 01 - Sensor Statechart

## Descripción
Implementación del modelo Sensor para 1 botón:

- BTN_A (B1 USER PushButton)

Se implementó debounce mediante máquina de estados.

## Estados implementados
- ST_BTN_UP
- ST_BTN_FALLING
- ST_BTN_DOWN
- ST_BTN_RISING

## Eventos implementados
- EV_BTN_UP
- EV_BTN_DOWN

## Configuración de hardware
Botón configurado como:
- GPIO input
- Pull-up interno
- Active Low

Pin utilizado:
- B1 USER = PC13



## Observaciones
Se verificó funcionamiento correcto del debounce.

Durante Live Variables, el estado DOWN no permanecía visible de forma estable.
