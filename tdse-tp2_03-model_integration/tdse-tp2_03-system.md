# TP2 - Actividad 03 - System Statechart

## Descripción
Implementación del modelo System para control de barrera automática.

## Estados implementados
- ST_SYS_WAIT_FOR_CAR_ARRIEVE
- ST_SYS_WAIT_FOR_BUTTON_PRESSED
- ST_SYS_WAIT_FOR_BARRIER_OPENED
- ST_SYS_WAIT_FOR_CAR_LEAVES
- ST_SYS_WAIT_FOR_BARRIER_CLOSED

## Eventos implementados
- EV_SYS_CAMERA
- EV_SYS_BUTTON
- EV_SYS_SENSOR_COIL

## Funcionamiento
Secuencia implementada:

1. Detección de auto por cámara
2. Espera pulsación botón
3. Apertura barrera
4. Espera salida del vehículo
5. Cierre barrera

## Observaciones
Se verificó correcto flujo entre estados y comunicación con task_actuator.