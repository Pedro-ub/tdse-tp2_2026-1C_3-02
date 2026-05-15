¡Hola! Este conjunto de archivos implementa la **lógica de lectura de un sensor (en este caso, un botón)** y un **mecanismo de comunicación entre tareas (Inter-Task Communication)** mediante una cola de eventos (FIFO). Es una arquitectura guiada por eventos (Event-Driven) muy común en sistemas embebidos profesionales.

A continuación, analizaremos el código paso a paso.

---

### 1. Análisis y Explicación de los Archivos

* **`task_sensor_attribute.h`:** Define las características de la tarea "Sensor". Declara los eventos físicos del botón (`EV_BTN_UP`, `EV_BTN_DOWN`), sus estados lógicos (`ST_BTN_IDLE`, `ST_BTN_ACTIVE`) y dos estructuras fundamentales:
    * `task_sensor_cfg_t`: La **configuración estática** (puerto de hardware, pin, qué nivel de voltaje significa "presionado", y qué señales lógicas debe emitir hacia el sistema).
    * `task_sensor_dta_t`: Los **datos dinámicos** o de tiempo de ejecución (estado actual de la máquina de estados, el último evento detectado y un contador de ticks).
* **`task_system_attribute.h`:** Define los atributos para la tarea principal "Sistema". Principalmente, declara los eventos que el sistema puede recibir (`EV_SYS_IDLE`, `EV_SYS_ACTIVE`) y sus estados.
* **`task_system_interface.c`:** Implementa una **Cola Circular (FIFO - First In, First Out)** segura. Sirve como "buzón de mensajes". Cuando el sensor detecta un cambio, no llama directamente a la tarea del sistema (lo cual acoplaría el código), sino que "deposita" un evento en esta cola mediante `put_event_task_system()`. Luego, el sistema podrá leer estos eventos a su propio ritmo con `get_event_task_system()`.
* **`task_sensor.c`:** Contiene la lógica ejecutable del sensor. Inicializa las variables, lee el hardware (el pin del microcontrolador) e implementa la máquina de estados finitos (FSM) que decide cuándo el botón fue presionado o soltado.

---

### 2. Evolución de Variables del Sensor

Desde la llamada a `task_sensor_init()` y en las sucesivas vueltas del loop (`task_sensor_update()`):

* **`index`:** Evoluciona iterando secuencialmente. Empieza en `0` y sube hasta `SENSOR_DTA_QTY - 1` (que en este caso es 0 porque solo hay 1 botón configurado). Permite que la misma función maneje múltiples botones si se agregan más a la lista de configuración.
* **`task_sensor_dta_list[index].tick`:** Su unidad de medida son **Ticks del sistema (milisegundos, ms)**. Al inicio no se inicializa explícitamente. *Nota técnica sobre este código específico:* A diferencia de algoritmos con anti-rebote (debounce) donde esta variable decrece o crece activamente, en el código actual suministrado el `tick` solo se asigna a `DEL_BTN_MIN` (0) si la máquina cae en el caso `default`. Es decir, en esta versión simplificada del código, el tiempo no gobierna las transiciones.
* **`task_sensor_dta_list[index].state`:**
    * *Inicio:* Se fuerza a `ST_BTN_IDLE` en el `init`.
    * *Evolución:* Cambia a `ST_BTN_ACTIVE` solo en la iteración donde se detecta que el botón bajó. Retorna a `ST_BTN_IDLE` en la iteración donde el botón subió.
* **`task_sensor_dta_list[index].event`:**
    * *Inicio:* Se inicializa en `EV_BTN_UP`.
    * *Evolución:* En cada vuelta del loop (`update`), evalúa instantáneamente el nivel del pin (`HAL_GPIO_ReadPin`). Si el nivel coincide con el de "presionado", se convierte en `EV_BTN_DOWN`; de lo contrario, vuelve a `EV_BTN_UP`. Oscila constantemente según el estado físico del botón.

---


### 3. Comportamiento de `task_sensor_statechart(uint32_t index)`

Esta función es el cerebro del sensor y hace dos cosas fundamentales en cada ejecución continua:

1.  **Mapeo de Hardware a Evento Lógico:** Compara el estado físico actual del pin (`HAL_GPIO_ReadPin`) con el estado de configuración de hardware (`pressed`). Traduce ese voltaje físico en un evento de software: `EV_BTN_DOWN` o `EV_BTN_UP`.
2.  **Evaluación de la Máquina de Estados (Switch-Case):**
    * Si está en **`ST_BTN_IDLE`** (botón suelto) y ve el evento **`EV_BTN_DOWN`**: Significa que el usuario *acaba de presionar* el botón. La función despacha el evento de sistema correspondiente (`EV_SYS_ACTIVE`) hacia la cola del sistema usando `put_event_task_system()`. Luego, cambia su propio estado a `ST_BTN_ACTIVE`.
    * Si está en **`ST_BTN_ACTIVE`** (botón presionado) y ve el evento **`EV_BTN_UP`**: Significa que el usuario *acaba de soltar* el botón. Despacha el evento `EV_SYS_IDLE` a la cola y vuelve al estado `ST_BTN_IDLE`.

Este diseño detecta **flancos** (el momento exacto del cambio) en lugar de niveles continuos, enviando un único mensaje a la cola por cada vez que se aprieta o se suelta el botón.

---

### 4. Evolución de las Variables de la Cola de Eventos (System Queue)

Esta cola amortigua los mensajes. Sus variables dentro de `task_system_interface.c` evolucionan así:

* **En el Inicio (suponiendo que se llama a `init_event_task_system()`):**
    * `event_task_system_queue.head = 0` (Índice donde se escribirá el próximo evento).
    * `event_task_system_queue.tail = 0` (Índice de donde se leerá el próximo evento).
    * `event_task_system_queue.count = 0` (Cantidad de eventos pendientes de lectura).
    * `event_task_system_queue.queue[i] = EMPTY` (Todo el arreglo, de 0 a 15, se llena con el valor 255).

* **En sucesivas ejecuciones (`update`), si se PRESIONA el botón:**
    * El statechart llama a `put_event_task_system(EV_SYS_ACTIVE)`.
    * La variable **`count`** suma 1 (ahora vale 1).
    * El evento `EV_SYS_ACTIVE` se guarda en **`queue[head]`** (es decir, en `queue[0]`).
    * La variable **`head`** avanza a 1. (Si `head` llegara a 16, volvería automáticamente a 0 por el comportamiento circular).

* **Si nadie lee la cola y el botón se SUELTA:**
    * Se llama a `put_event_task_system(EV_SYS_IDLE)`.
    * **`count`** pasa a valer 2.
    * `EV_SYS_IDLE` se guarda en **`queue[1]`**.
    * **`head`** avanza a 2.
    * **`tail`** sigue valiendo 0 (porque la tarea del sistema todavía no ha llamado a `get_event_task_system()` para procesar estos mensajes).

En resumen, la `head` avanza "persiguiendo" a la `tail` cada vez que el sensor actúa, y la `tail` avanzará cada vez que la aplicación principal procese dichos eventos.