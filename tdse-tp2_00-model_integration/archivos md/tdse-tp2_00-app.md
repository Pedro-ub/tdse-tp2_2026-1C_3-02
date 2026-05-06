¡Con mucho gusto! Este conjunto de archivos implementa un sistema clásico de **planificación de tareas (scheduler) bare-metal con análisis de rendimiento (profiling)**. 

Básicamente, el código no solo ejecuta tus funciones periódicamente, sino que también mide con altísima precisión cuánto tiempo tarda la CPU en ejecutar cada una de ellas.

A continuación, desglosamos el funcionamiento de cada archivo y la evolución de las variables.

---

## 1. Análisis de los Archivos Fuente

* **`dwt.h` (Data Watchpoint and Trace):** Este archivo maneja el contador de ciclos DWT del procesador ARM Cortex-M. El DWT es un periférico de hardware que incrementa un registro (`CYCCNT`) por cada ciclo de reloj del sistema. Es la herramienta principal para medir el tiempo con precisión de sub-microsegundos, lo que permite cronometrar cuánto tarda exactamente un bloque de código en ejecutarse.
* **`systick.c`:** Configura y maneja el temporizador del sistema (SysTick). Normalmente, genera una interrupción periódica (ej. cada 1 milisegundo) que sirve como la base de tiempo macroscópica de la aplicación, llevando la cuenta del "tiempo real" desde que arrancó el sistema.
* **`logger.c` y `logger.h`:** Conforman el módulo de registro de eventos (logging). Proveen macros (como `LOGGER_INFO()`) y funciones para formatear cadenas de texto (similar a `printf`) y enviarlas hacia el exterior, comúnmente a través de un puerto serie (UART) o mediante Semihosting para ver los mensajes en la consola de depuración.
* **`app.c`:** Es el corazón de la aplicación. Contiene el despachador de tareas (task dispatcher). En lugar de tener el código "suelto" en un `while(1)`, el código se organiza en una lista o arreglo de tareas (`task_dta_list`). La función `app_init()` inicializa las variables de estadísticas, y `app_update()` recorre las tareas ejecutándolas secuencialmente y midiendo sus tiempos mediante el DWT.

---

## 2. Evolución de las Variables

Durante el inicio (`app_init()`) y la ejecución continua del bucle (`app_update()`), las variables evolucionan para generar un perfil estadístico del sistema.

| Variable | Unidad | Evolución |
| :--- | :--- | :--- |
| **`g_app_tick_cnt`** | **ms** (milisegundos) o Ticks | Se inicializa en 0 al arrancar. Se incrementa automáticamente en 1 cada vez que ocurre una interrupción del SysTick (generalmente cada 1 ms). Es un contador monótono ascendente del tiempo de vida del sistema. |
| **`g_app_runtime_us`** | **µs** (microsegundos) | Representa el tiempo total de ejecución. Se calcula leyendo el contador de ciclos del DWT y dividiéndolo por la frecuencia del reloj en MHz. Inicializa en 0 y crece continuamente a medida que el microcontrolador ejecuta instrucciones. |
| **`index`** | Adimensional | Es la variable iteradora dentro de `app_update()`. Arranca en 0 y sube hasta la cantidad total de tareas configuradas menos 1. Al terminar de recorrer todas las tareas en un ciclo, vuelve a 0 en la siguiente llamada a `app_update()`. |
| **`task_dta_list[index].NOE`** | Adimensional (veces) | *Number Of Executions*. Inicializa en 0 en `app_init()`. Cada vez que el bucle pasa por la tarea `index` y la ejecuta, este contador suma +1. |
| **`task_dta_list[index].LET`** | **µs** (microsegundos) | *Last Execution Time*. El tiempo que tardó la tarea en su última ejecución. Se calcula restando el valor del DWT antes y después de llamar a la tarea. Cambia (oscila) en cada pasada según el camino lógico que haya tomado la función (ej. si entró a un `if` o no). |
| **`task_dta_list[index].BCET`** | **µs** (microsegundos) | *Best Case Execution Time*. En `app_init()` se inicializa con un valor artificialmente alto (el máximo posible). Luego, en cada ciclo, si el `LET` actual es **menor** que el `BCET` guardado, se actualiza. Retiene el tiempo más rápido registrado históricamente para esa tarea. |
| **`task_dta_list[index].WCET`** | **µs** (microsegundos) | *Worst Case Execution Time*. En `app_init()` se inicializa en 0. Si el `LET` actual es **mayor** que el `WCET` guardado, se actualiza. Retiene el pico máximo de tiempo que la tarea ha bloqueado al procesador. |

---

## 3. El Impacto de usar `LOGGER_INFO()`

El uso de funciones de loggeo en sistemas embebidos tiene un costo computacional altísimo, especialmente si la transmisión de datos es bloqueante (esperar a que el hardware UART termine de enviar byte por byte) o requiere formateo complejo de strings.

* **Impacto en `task_dta_list[index].WCET`:**
    Si dentro de una tarea agregas un `LOGGER_INFO()`, el tiempo de ejecución de esa tarea se disparará enormemente durante esa iteración específica. Por ejemplo, una tarea de lectura de sensor que toma 5 µs podría saltar a 2000 µs por culpa del `LOGGER_INFO()`. 
    Ese nuevo `LET` gigantesco reemplazará inmediatamente el valor anterior de **`WCET`** (Worst Case Execution Time). Tu "peor caso" ya no será representativo de tu lógica matemática o de control, sino que estará dominado por el tiempo de transmisión del mensaje de log.

* **Impacto en `g_app_runtime_us`:**
    Al tardar mucho más tiempo procesando las tareas debido al bloqueo de las impresiones en consola, el bucle general se retrasa. La CPU se gasta procesando cadenas de caracteres en lugar de hacer control. Por lo tanto, `g_app_runtime_us` crecerá a la misma velocidad física (el tiempo de reloj real avanza igual), pero en ese mismo lapso de tiempo real, tu sistema habrá dado muchas **menos iteraciones** o vueltas de control en `app_update()`. La eficiencia y responsividad del sistema decaen drásticamente.