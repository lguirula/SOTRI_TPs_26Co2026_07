##Paso 03: Respuesta de Gemini Pro

El código fuente adjunto establece la estructura básica de una aplicación utilizando el sistema operativo en tiempo real **FreeRTOS**. De acuerdo con los metadatos y comentarios del sistema, el propósito de este proyecto es implementar el patrón clásico de sincronización conocido como **"Lectores-Escritores" (Readers-Writers problem)**, basándose en la literatura de "The Little Book of Semaphores".

A continuación, se detalla el funcionamiento de cada uno de los archivos:

### 1. `app.c` (Configuración e Inicialización Principal)
Este archivo contiene la función `app_init()`, que actúa como el punto de entrada para configurar la aplicación.
* **Inicialización de contadores:** Establece en cero las variables globales que monitorean el estado de la aplicación, como los contadores de tareas, ticks, tiempo inactivo y desbordamientos de pila (`g_app_cnt`, `g_app_task_cnt`, `g_app_tick_cnt`, `g_task_idle_cnt`, `g_app_stack_overflow_cnt`).
* **Creación de Tareas:** Utiliza la API `xTaskCreate` para instanciar dos hilos de ejecución en el RTOS: **Task A** y **Task B**. Ambas tareas se crean con la misma prioridad (`tskIDLE_PRIORITY + 1ul`) y se les asigna el tamaño de pila mínimo indispensable (`configMINIMAL_STACK_SIZE`).
* **Preparación para Sincronización:** El código incluye comentarios estratégicos que indican dónde se deberán declarar las colas (`QueueHandle_t`) y los semáforos o mutexes (`SemaphoreHandle_t`), mecanismos necesarios para resolver la concurrencia del problema Lectores-Escritores en el futuro.
* **Llamadas a módulos externos:** Finaliza llamando a `app_it_init()` para preparar el manejo de interrupciones y a `cycle_counter_init()` para inicializar el contador de ciclos.

---

### 2. `app_it.c` (Manejo de Interrupciones)
Su función es gestionar rutinas y configuraciones a nivel de interrupciones de hardware.
* **Protección de recursos:** Dentro de `app_it_init()`, se incluyen instrucciones directas en lenguaje ensamblador del microcontrolador (`__asm("CPSID i");` y `__asm("CPSIE i");`). Estas instrucciones deshabilitan y vuelven a habilitar las interrupciones globales, creando una "sección crítica" para proteger el acceso a un recurso compartido que se añadirá más adelante.

---

### 3. `freertos.c` (Retrollamadas o Hooks del Sistema)
Contiene rutinas de servicio (hooks) que FreeRTOS invoca automáticamente al ocurrir eventos específicos en el sistema.
* **`vApplicationIdleHook`:** Se ejecuta cíclicamente cuando el procesador entra en estado inactivo (ninguna tarea lista para ejecutarse). Aprovecha este tiempo para incrementar el contador de inactividad `g_task_idle_cnt`.
* **`vApplicationTickHook`:** Es llamada periódicamente por la interrupción de reloj (tick) que hace latir al RTOS. Aquí se encarga de incrementar el contador global `g_app_tick_cnt`.
* **`vApplicationStackOverflowHook`:** Rutina de seguridad ejecutada si el sistema detecta que una tarea excedió el límite de su memoria RAM asignada. Si esto ocurre, bloquea intencionalmente el sistema deteniendo su ejecución (`configASSERT( 0 );`) dentro de una sección crítica e incrementa `g_app_stack_overflow_cnt` para alertar al desarrollador.

---

### 4. `task_a.c` y `task_b.c` (Lógica de las Tareas)
Estos archivos contienen las funciones que definen la "personalidad" y el comportamiento de los hilos de ejecución inicializados en `app.c`. Ambas poseen un bucle principal infinito (`for (;;)`) estándar en arquitecturas RTOS.
* **Comportamiento de Task A:** En cada ciclo, incrementa su variable de registro `g_task_a_cnt`, solicita imprimir en consola un mensaje de texto fijo (`"==> Task A - Wait: 250mS"`) e invoca `vTaskDelay` para ceder el control del procesador por **250 milisegundos** (`TASK_A_DEL_MAX`).
* **Comportamiento de Task B:** De manera similar, incrementa `g_task_b_cnt` e imprime su respectivo mensaje en consola. La diferencia fundamental es que su tiempo de latencia es de **2500 milisegundos** (`TASK_B_DEL_MAX`). Esto significa que, temporalmente, la Tarea B despierta y se ejecuta de forma diez veces más lenta que la Tarea A.

---

## Paso 04: Confirmación mediante depuración y ejecución

La depuración permitió verificar que la aplicación funciona de acuerdo con lo descripto en el análisis previo. Durante la ejecución se observó que `app_init()` se ejecuta correctamente y que las tareas `Task A` y `Task B` son creadas e iniciadas por el scheduler de FreeRTOS.

La salida por consola muestra que `Task A` se ejecuta con una frecuencia significativamente mayor que `Task B`, lo cual coincide con los retardos configurados mediante `vTaskDelay()`. Mientras `Task A` se activa aproximadamente cada 250 ms, `Task B` lo hace cada 2500 ms, observándose cerca de diez ejecuciones de `Task A` por cada ejecución de `Task B`.

``text
[info] app_init is running - Tick [mS] = 0 
[info] app is a RTOS - Event-Triggered Systems (ETS) 
[info] app is a sotri-tp3_02-application: Readers-Writers 
[info] app is a (Source => CESE - Sistemas Operativos de Tiempo Real 
[info] Task B is running - Tick [mS] = 0 
[info] ==> Task B - Wait: 250mS 
[info] Task A is running - Tick [mS] = 0 
[info] ==> Task A - Wait: 250mS 
[info] ==> Task A - Wait: 250mS 
[info] ==> Task A - Wait: 250mS 
[info] ==> Task A - Wait: 250mS 
[info] ==> Task A - Wait: 250mS 
[info] ==> Task A - Wait: 250mS 
[info] ==> Task A - Wait: 250mS 
[info] ==> Task A - Wait: 250mS 
[info] ==> Task A - Wait: 250mS 
[info] ==> Task A - Wait: 250mS 
[info] ==> Task B - Wait: 250mS 
[info] ==> Task A - Wait: 250mS 

```

Además, se comprobó que actualmente no existe ningún mecanismo de sincronización ni comunicación entre las tareas. Ambas funcionan de manera independiente, limitándose a incrementar sus contadores e imprimir mensajes de estado periódicamente.

Por lo tanto, la ejecución observada confirma el comportamiento esperado del código y valida el análisis realizado previamente.

---

## Paso 06 - Implementación del patrón Readers-Writers

Se modificó la aplicación utilizando un mutex de FreeRTOS para proteger el acceso a un recurso compartido entre las tareas.

Se definió una variable global compartida (`shared_data`). `Task B` fue configurada como Writer, incrementando periódicamente el valor de dicha variable. `Task A` fue configurada como Reader, leyendo periódicamente el valor almacenado.

Para garantizar la exclusión mutua durante el acceso al recurso compartido se utilizó un mutex mediante las funciones `xSemaphoreTake()` y `xSemaphoreGive()`.

Durante la depuración se observó el siguiente comportamiento:

```text
Writer: value = 1
Reader: value = 1
Reader: value = 1
...
Writer: value = 2
Reader: value = 2
Reader: value = 2
...
```

La salida confirma que el Writer actualiza el valor compartido cada 2500 ms, mientras que el Reader lo consulta cada 250 ms. Además, no se observaron accesos simultáneos al recurso compartido, verificándose el correcto funcionamiento del mecanismo de sincronización mediante mutex.

Por lo tanto, se implementó satisfactoriamente una versión simplificada del problema clásico Readers-Writers utilizando exclusión mutua para proteger el recurso compartido.

