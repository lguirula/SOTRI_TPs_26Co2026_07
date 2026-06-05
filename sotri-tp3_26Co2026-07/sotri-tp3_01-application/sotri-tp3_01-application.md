## Paso 06: Analisis de Gemini Pro

El código fuente proporcionado constituye la estructura base (un esqueleto) para una aplicación de un Sistema Operativo en Tiempo Real (RTOS), específicamente FreeRTOS. Según los comentarios de la cabecera, el objetivo principal del proyecto es implementar el problema clásico de concurrencia conocido como **"Productor-Consumidor"**. 

A continuación, se presenta un análisis detallado del funcionamiento de cada archivo:

### 1. `app.c` (Configuración principal)
Este archivo es el núcleo de la configuración de la aplicación.
* **Inicialización:** La función `app_init()` se encarga de arrancar el entorno. Inicia variables globales que sirven como contadores de diagnóstico (ticks, tareas, inactividad, etc.).
* **Creación de tareas:** Aquí se le indica al RTOS que cree dos hilos de ejecución utilizando `xTaskCreate`:
    * **Task A**.
    * **Task B**.
* **Prioridades:** Ambas tareas son creadas con la misma prioridad base (`tskIDLE_PRIORITY + 1`) y se les asigna un tamaño mínimo de pila de memoria.
* **Marcadores de posición:** El código incluye comentarios indicando dónde se deben declarar las colas (QueueHandle) y los semáforos (SemaphoreHandle o mutex) para sincronizar a los hilos en el futuro, lo cual es necesario para el patrón Productor-Consumidor, pero dichas variables no están operativas todavía.

---

### 2. `app_it.c` (Manejo de Interrupciones)
Este archivo está diseñado para gestionar las interrupciones específicas de la aplicación.
* **Estado actual:** Posee la función `app_it_init()` que actualmente funciona como un cascarón o plantilla.
* **Protección básica:** Contiene instrucciones en lenguaje ensamblador (`__asm("CPSID i");` y `__asm("CPSIE i");`) que deshabilitan y vuelven a habilitar las interrupciones del procesador. Esto es un patrón clásico para crear una "sección crítica" y proteger un recurso compartido, aunque el recurso en sí aún no está implementado.

---

### 3. `freertos.c` (Hooks o Retrollamadas del Sistema)
Contiene funciones especiales llamadas *hooks* que el sistema operativo FreeRTOS ejecuta automáticamente cuando ocurren eventos específicos a nivel de sistema.
* **`vApplicationIdleHook`:** Se invoca cuando el procesador no tiene ninguna tarea activa que ejecutar (tiempo inactivo). En este código, aprovecha ese tiempo para incrementar un contador de inactividad (`g_task_idle_cnt`).
* **`vApplicationTickHook`:** Se ejecuta con cada "tick" o interrupción del reloj del sistema operativo. Aquí se incrementa el contador general de la aplicación (`g_app_tick_cnt`).
* **`vApplicationStackOverflowHook`:** Es una medida de seguridad. Si el sistema detecta que una tarea excedió el límite de su memoria asignada (desbordamiento de pila), esta función detiene la ejecución del sistema intencionalmente (`configASSERT( 0 );`) para permitir depurar la falla e incrementa el contador `g_app_stack_overflow_cnt`.

---

### 4. `task_a.c` y `task_b.c` (Lógica de las Tareas)
Estos archivos definen el comportamiento de las dos tareas creadas en `app.c`. Tienen una estructura idéntica, pero con temporizaciones diferentes.
* **Estructura de bucle:** Ambas funciones (`task_a` y `task_b`) entran en un bucle infinito `for (;;)` que es el comportamiento estándar de un hilo de vida continua en un RTOS.
* **Comportamiento de Task A:** Incrementa su contador individual, imprime un mensaje de estado usando `LOGGER_INFO` y luego suspende su ejecución (liberando el procesador) durante **250 milisegundos** (`TASK_A_DEL_MAX`) mediante la función `vTaskDelay`.
* **Comportamiento de Task B:** Incrementa su contador individual y suspende su ejecución durante **2500 milisegundos** (`TASK_B_DEL_MAX`). *(Nota técnica: Aunque el tiempo de espera real es de 2500 ms, la variable de texto que imprime en el log se llama `p_task_b_wait_250mS` indicando erróneamente un tiempo de 250mS).*

---

### Resumen del flujo
El código configura el sistema operativo, habilita su monitoreo (mediante los *hooks*) y pone a correr dos procesos independientes. **Task A** despierta, trabaja (imprime un log) y duerme de forma rápida, mientras que **Task B** realiza el mismo ciclo de forma mucho más lenta. Aún no existe transferencia de datos o sincronización (semáforos/colas) entre ellas, por lo que el patrón Productor-Consumidor se encuentra en una etapa de preparación arquitectónica.

---

## Paso 07: Confirmación mediante depuración y ejecución

La depuración permitió confirmar el funcionamiento general descrito en el análisis previo. Al iniciar el sistema se ejecuta correctamente la función `app_init()`, lo cual se verifica mediante los primeros mensajes mostrados por el logger:

```text
[info] app_init is running - Tick [mS] = 0
[info] app is a RTOS - Event-Triggered Systems (ETS)
```

También se confirmó que las tareas `Task A` y `Task B` son creadas exitosamente y comienzan a ejecutarse cuando se inicia el scheduler de FreeRTOS.

Durante la ejecución se observó que `Task A` se ejecuta con mayor frecuencia que `Task B`, debido a que poseen distintos tiempos de retardo mediante `vTaskDelay()`. Analizando el código fuente se verificó que:

```c
#define TASK_A_DEL_MAX (pdMS_TO_TICKS(250ul))
#define TASK_B_DEL_MAX (pdMS_TO_TICKS(2500ul))
```

Por lo tanto, `Task A` se ejecuta aproximadamente cada 250 ms, mientras que `Task B` lo hace cada 2500 ms. Esto coincide con la salida observada, donde aparecen cerca de diez ejecuciones de `Task A` por cada ejecución de `Task B`.

Se detectó además una inconsistencia en el mensaje mostrado por `Task B`, ya que imprime:

```text
==> Task B - Wait: 250mS
```

aunque el retardo real configurado es de 2500 ms.

Finalmente, se confirmó que la aplicación aún no implementa el patrón Productor-Consumidor, ya que no se utilizan colas, semáforos ni mutexes para la comunicación o sincronización entre tareas. Actualmente ambas tareas funcionan de forma independiente y únicamente generan mensajes de estado.

### Conclusión

La depuración permitió verificar que:

- `app_init()` se ejecuta correctamente.
- Las tareas `Task A` y `Task B` son creadas e iniciadas por FreeRTOS.
- El scheduler funciona correctamente.
- Los retardos configurados mediante `vTaskDelay()` se cumplen.
- Existe un error únicamente en el mensaje de log de `Task B`.
- Aún no hay comunicación ni sincronización entre tareas.

---
## Paso 09 - Implementación del patrón Productor-Consumidor

Se modificó la sincronización entre `Task A` y `Task B` utilizando un semáforo binario de FreeRTOS.

Se creó un semáforo binario compartido entre ambas tareas. `Task A` fue configurada como Productor y libera el semáforo mediante `xSemaphoreGive()` cada vez que genera un evento. Por su parte, `Task B` fue configurada como Consumidor y permanece bloqueada mediante `xSemaphoreTake(portMAX_DELAY)` hasta que el Productor libera el semáforo.

Durante la depuración se observó que la salida por consola cambió respecto de la versión anterior. Mientras que inicialmente ambas tareas se ejecutaban de forma independiente según sus temporizaciones, luego de incorporar el semáforo se verificó la siguiente secuencia:

```text
[info]  
[info] app_init is running - Tick [mS] = 0
[info]  app is a RTOS - Event-Triggered Systems (ETS)
[info]  app is a sotri-tp3_01-application: Producer-Consumer
[info]  app is a (Source => CESE - Sistemas Operativos de Tiempo Real
[info]  
[info]   Task B is running - Tick [mS] = 0
[info]  
[info]   Task A is running - Tick [mS] = 0
[info] Producer: produce item
[info] Consumer: consume item
[info] Producer: produce item
[info] Consumer: consume item
[info] Producer: produce item
[info] Consumer: consume item

```

Este comportamiento confirma que `Task B` ya no se ejecuta periódicamente por temporización propia, sino únicamente cuando recibe la señal enviada por `Task A`.

Por lo tanto, se verificó experimentalmente el correcto funcionamiento del patrón clásico Productor-Consumidor utilizando sincronización mediante semáforos binarios.
