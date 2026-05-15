# TP2 - Actividad 05 - 2 Actuator Statechart

## Descripción
Implementación del modelo Actuator para 2 LEDs externos:

- LED_BARRIER_OPEN
- LED_BARRIER_CLOSE

## Estados implementados
- ST_LED_OFF
- ST_LED_ON
- ST_LED_BLINK

## Eventos implementados
- EV_LED_OFF
- EV_LED_ON
- EV_LED_BLINK

## Configuración hardware
LEDs externos:
- GPIO output push-pull
- Active High
- Resistencias limitadoras de corriente

## Funcionamiento
- LED_BARRIER_CLOSE inicia encendido
- LED_BARRIER_OPEN inicia apagado
- Durante transición se utiliza blinking
- Al finalizar transición queda fijo

## Observaciones
Se verificó correcto control independiente de ambos actuadores.