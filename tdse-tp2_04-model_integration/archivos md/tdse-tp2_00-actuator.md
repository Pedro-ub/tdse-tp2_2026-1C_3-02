¡Hola! Para completar el ciclo de esta arquitectura guiada por eventos, analizaremos los archivos correspondientes al **Actuador** (en este caso, un LED). 

A continuación, analizaremos el funcionamiento y la evolución de las variables de este módulo.

---

### 1. Análisis y Explicación de los Archivos

* **`task_actuator_attribute.h`**: Define el diccionario de datos para el actuador.
    * Declara los **eventos** que puede recibir (`EV_LED_IDLE` para apagar, `EV_LED_ACTIVE` para encender).
    * Declara los **estados** posibles de su máquina de estados (`ST_LED_IDLE`, `ST_LED_ACTIVE`).
    * Define las estructuras `task_actuator_cfg_t` (configuración constante de hardware: puerto, pin, niveles de voltaje para encendido/apagado) y `task_actuator_dta_t` (variables de ejecución: estado, evento, flag y tick).
* **`task_actuator_interface.c`**: Es el canal de comunicación. A diferencia del Sistema (que usa una cola FIFO), el Actuador utiliza una **comunicación directa por buzón de un solo mensaje**. La función `put_event_task_actuator()` sobrescribe directamente la variable `event` del actuador destino y levanta una bandera (`flag = true`) para avisarle que tiene una orden nueva.
* **`task_actuator.c`**: Es el motor del actuador. Contiene la inicialización (`task_actuator_init`), el bucle que recorre todos los actuadores (`task_actuator_update`) y la máquina de estados que interactúa con el hardware real del microcontrolador para encender o apagar el LED (`task_actuator_statechart`).

---

### 2. Evolución de Variables Internas del Actuador

Desde el inicio (`task_actuator_init`) y durante la ejecución continua de `task_actuator_update`:

* **`index`**: Es el iterador del bucle `for` en el `update`. Comienza en `0` y sube hasta `ACTUATOR_DTA_QTY - 1`. Sirve para actualizar la máquina de estados de todos los actuadores que tengas configurados (ej. si tuvieras 3 LEDs, iteraría 0, 1, 2).
* **`task_actuator_dta_list[index].tick`**: Se mide en **Ticks del sistema (milisegundos, ms)**. Generalmente se inicializa en 0. Podría usarse si el actuador tuviera un estado de "parpadeo" (Blinking) para contar el tiempo de encendido/apagado automático, pero en un control básico (On/Off) esta variable no afecta las transiciones.
* **`task_actuator_dta_list[index].state`**:
    * *Inicio*: Se inicializa forzosamente en `ST_LED_IDLE` (LED apagado).
    * *Evolución*: Cambiará a `ST_LED_ACTIVE` cuando la máquina de estados procese el comando de encendido, y volverá a `ST_LED_IDLE` cuando procese el comando de apagado.
* **`task_actuator_dta_list[index].event`**:
    * *Inicio*: Se inicializa en `EV_LED_IDLE`.
    * *Evolución*: Permanece estático hasta que un ente externo (la Tarea del Sistema) lo sobrescribe a través de la interfaz.
* **`task_actuator_dta_list[index].flag`**:
    * *Inicio*: Se inicializa en `false`.
    * *Evolución*: Permanece en `false` mientras no hay comandos nuevos. Cuando el Sistema envía un comando, cambia a `true` (vía la interfaz). Apenas el `statechart` procesa ese comando, vuelve a bajar la bandera (`false`).

---

### 3. Comportamiento de `task_actuator_statechart(uint32_t index)`

Esta función es la que realmente enciende y apaga la luz actuando sobre los pines físicos (`HAL_GPIO_WritePin`). Su comportamiento mediante un `switch-case` es el siguiente:

1. Lee su estado actual, su evento y su `flag`.
2. **Si el estado es `ST_LED_IDLE`:** Evalúa si hay una orden pendiente (`flag == true`) y si la orden es encender (`event == EV_LED_ACTIVE`).
    * Si ambas son ciertas: Baja la bandera (`flag = false`), escribe en el pin de hardware el nivel correspondiente a `led_on` (configurado en `cfg_t`), y transiciona su estado a `ST_LED_ACTIVE`.
3. **Si el estado es `ST_LED_ACTIVE`:** Evalúa si hay una orden pendiente (`flag == true`) y si la orden es apagar (`event == EV_LED_IDLE`).
    * Si ambas son ciertas: Baja la bandera (`flag = false`), escribe en el pin de hardware el nivel correspondiente a `led_off`, y transiciona su estado de regreso a `ST_LED_IDLE`.

---

### 4. Evolución de Variables de la Interfaz (`put_event_task_actuator`)

Cuando la Tarea del Sistema decide accionar el LED, llama a la función `put_event_task_actuator(event, identifier)` (en `task_actuator_interface.c`). Así evolucionan sus variables en ese instante:

* **`identifier`**: No evoluciona internamente, es el parámetro de entrada (ej. `ID_LED_A`, que es 0). Se usa directamente como índice (`identifier`) para localizar a qué actuador específico se le enviará el mensaje dentro del arreglo `task_actuator_dta_list`.
* **`task_actuator_dta_list[identifier].event`**: Es sobrescrita en el acto con el valor pasado por argumento (ej. pasa a valer `EV_LED_ACTIVE`).
* **`task_actuator_dta_list[identifier].flag`**: Pasa instantáneamente de `false` a `true`. 

**Nota de diseño importante:** Como esta interfaz usa una sobrescritura directa y no una cola (Queue), si el Sistema le manda tres comandos al Actuador de forma extremadamente rápida (antes de que el Actuador logre ejecutar su propio `update`), los dos primeros comandos se perderán (se sobreescribirán) y el Actuador solo verá y ejecutará el **último** evento enviado. Esto es ideal para un LED, donde solo importa el estado final deseado.