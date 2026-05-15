¡Hola! Este conjunto de archivos implementa el "cerebro" o la **lógica central (Sistema)** de la aplicación, así como los canales de comunicación hacia los actuadores (ej. un LED). 

En esta arquitectura guiada por eventos, la tarea del "Sistema" actúa como un intermediario: recibe los eventos generados por el sensor (botón presionado/suelto) desde una cola, procesa esos datos mediante su propia máquina de estados, y finalmente envía comandos al actuador.



A continuación, analizaremos los archivos y la evolución de sus variables.

---

### 1. Análisis de los Archivos Fuente

* **`task_system_attribute.h`**: Define el vocabulario de la tarea del Sistema. Declara los eventos que el sistema entiende (`EV_SYS_IDLE`, `EV_SYS_ACTIVE`), sus posibles estados lógicos (`ST_SYS_IDLE`, `ST_SYS_ACTIVE`) y la estructura de datos que mantendrá su contexto en memoria (`task_system_dta_t`).
* **`task_actuator_attribute.h`**: Similar al anterior, pero para el Actuador (el LED). Define eventos (`EV_LED_IDLE`, `EV_LED_ACTIVE`), estados y estructuras necesarias para controlar salidas físicas.
* **`task_system_interface.c`**: Contiene la implementación de la **Cola Circular de Eventos del Sistema**. Es el "buzón de entrada". Otras tareas (como el sensor) usan este archivo para depositar mensajes (`put_event...`) y el sistema lo usa para leerlos (`get_event...`).
* **`task_actuator_interface.c`**: Es el "buzón de salida" del sistema hacia el actuador. A diferencia del sistema que usa una cola (FIFO), la función `put_event_task_actuator()` implementada aquí escribe directamente sobre las variables de la tarea del actuador, sobrescribiendo cualquier evento anterior.
* **`task_system.c`**: Es la lógica de control. Consume los eventos de su buzón de entrada, evalúa su máquina de estados (`task_system_statechart`) y decide qué acciones tomar (como enviar un mensaje al actuador).

---

### 2. Evolución de las Variables del Sistema

Desde el inicio (`task_system_init`) y durante el bucle (`task_system_update`):

* **`index`**: Es el iterador de tareas. Comienza en 0 y llega hasta `SYSTEM_DTA_QTY - 1`. Permite gestionar múltiples instancias de "sistemas" o "modos" si el proyecto crece.
* **`task_system_dta_list[index].tick`**: Su unidad son los **milisegundos (ms) / Ticks del sistema**. En este código particular, no gobierna transiciones automáticas por tiempo, por lo que su valor se inicializa (generalmente en 0) pero no afecta la lógica primaria mostrada.
* **`task_system_dta_list[index].state`**: 
    * *Inicio*: Se fuerza a `ST_SYS_IDLE` en la inicialización.
    * *Evolución*: Cambiará a `ST_SYS_ACTIVE` cuando reciba y procese el evento `EV_SYS_ACTIVE`. Volverá a `ST_SYS_IDLE` cuando procese `EV_SYS_IDLE`.
* **`task_system_dta_list[index].event`**:
    * *Inicio*: Se inicializa típicamente en `EV_SYS_IDLE`.
    * *Evolución*: En cada ejecución de `update`, si hay eventos en la cola, esta variable toma el valor extraído de la cola mediante `get_event_task_system()`.
* **`task_system_dta_list[index].flag`**: 
    * *Inicio*: Se inicializa en `false`.
    * *Evolución*: Actúa como un aviso de "Evento nuevo no procesado". Se pone en `true` justo después de extraer un evento de la cola. Cuando la máquina de estados consume el evento y realiza la transición, lo vuelve a poner en `false`.

---

### 3. Comportamiento de `task_system_statechart(uint32_t index)`

Esta función actúa como un despachador de modos de funcionamiento.
1.  Evalúa la variable global `g_task_system_mode`.
2.  Si el modo es `NORMAL` (el caso por defecto), delega la ejecución a la función especializada **`task_system_normal_statechart()`**.
3.  Dentro de esa sub-función, primero se revisa si hay mensajes pendientes en la cola (`any_event_task_system()`). Si los hay, se extraen, se guardan en `.event` y se levanta la `.flag`.
4.  Luego, un `switch-case` evalúa el estado actual (`ST_SYS_IDLE` o `ST_SYS_ACTIVE`).
5.  Si está en `ST_SYS_IDLE`, hay un nuevo evento (`flag == true`), y el evento es `EV_SYS_ACTIVE`:
    * Baja la bandera (`flag = false`).
    * **Envía un comando al actuador**: `put_event_task_actuator(EV_LED_ACTIVE, ID_LED_A)`.
    * Cambia su propio estado a `ST_SYS_ACTIVE`.
6.  Hace lo opuesto si está en `ST_SYS_ACTIVE` y recibe un `EV_SYS_IDLE`.



---

### 4. Evolución de la Cola de Eventos (`event_task_system_queue`)

Estas variables de `task_system_interface.c` evolucionan desde la perspectiva de la **lectura** (consumo) por parte del sistema:

* **En el Inicio (`init_event_task_system`)**:
    * `i`: Iterador local que va de 0 a 15 (`QUEUE_LENGTH - 1`).
    * `queue[i]`: Todo el arreglo se llena con el valor `EMPTY` (generalmente 255).
    * `head`, `tail` y `count`: Se inicializan en 0.
* **En sucesivas ejecuciones (`update` del sistema)**:
    * Si la tarea del Sensor previamente insertó un evento, la cola tendrá `count > 0`.
    * El sistema llama a `get_event_task_system()`.
    * **`count`** disminuye en 1 (`count--`).
    * Se lee el evento guardado en **`queue[tail]`**.
    * **`tail`** incrementa en 1 (`tail++`), avanzando para apuntar al próximo evento sin leer. Si `tail` llega al final del arreglo (`QUEUE_LENGTH`), vuelve a 0 (comportamiento circular).

---

### 5. Evolución de Variables del Actuador vía Interfaz

Cuando el Sistema decide que el LED debe cambiar, llama a `put_event_task_actuator()` en `task_actuator_interface.c`. Sus variables evolucionan así:

* **`identifier`**: Es el argumento que se pasa a la función (ej. `ID_LED_A`). Se usa como índice (`identifier`) para saber a qué actuador específico de la lista (`task_actuator_dta_list`) apuntar. Su valor se mantiene constante para el LED A (0).
* **`task_actuator_dta_list[identifier].event`**: Se sobrescribe inmediatamente con el evento que manda el sistema (por ejemplo, pasa de `EV_LED_IDLE` a `EV_LED_ACTIVE`). A diferencia del sistema, aquí no hay cola; el último evento "pisa" al anterior.
* **`task_actuator_dta_list[identifier].flag`**: Pasa a valer `true`. Esto dejará marcado el evento como "pendiente" para que, cuando el bucle principal finalmente ejecute la actualización del Actuador (`task_actuator_update`), este sepa que tiene una nueva orden de encender o apagar el hardware.