# CESE - Sistemas Operativos de Tiempo Real

## Trabajo Practico N°: 3 - Sincronizacion de Tareas de FreeRTOS (Actividad 04)

> **Proyecto:** `sotri-tp3_04-application`  
> **Aplicacion:** `sotri-tp3_04-application: Security airlock` (puerta esclusa de seguridad)  
> **Placa:** NUCLEO-L4R5ZI (STM32L4R5ZI). La guia original referencia NUCLEO-F103RB; la arquitectura APP es equivalente.


### Paso 03 :

A continuación se analiza y explica el funcionamiento del código fuente contenido en los archivos solicitados:

* **app.c**: Contiene la inicialización principal de la aplicación mediante la función `app_init()`. En este archivo se crean todos los recursos de sincronización de FreeRTOS, como semáforos binarios y mutex (por ejemplo, usando `xSemaphoreCreateBinary()` y `xSemaphoreCreateMutex()`), y se añaden al registro para facilitar el *debugging*. Adicionalmente, se encarga de instanciar cada una de las tareas del sistema (`xTaskCreate`), asignándoles su función de entrada, nombre, tamaño de pila y su respectivo nivel de prioridad.

* **app_it.c**: Encargado del manejo de interrupciones específicas de la aplicación. Define la función `app_it_init()` para preparar las interrupciones del hardware y alberga la *callback* `HAL_GPIO_EXTI_Callback(uint16_t GPIO_Pin)` que se ejecuta automáticamente cuando se detecta un evento en los pines configurados con interrupciones externas (como un botón físico).

* **task_entry_a.c** (task_enrtry_a.c): Implementa el comportamiento de una tarea de entrada (por ejemplo, ingreso a una vía). Funciona mediante un bucle infinito que inicialmente se bloquea esperando un evento a través de un semáforo binario (`xSemaphoreTake`). Una vez que recibe la señal, utiliza un mutex (`xSemaphoreTake` sobre `h_mutex_mut_sem`) para acceder de forma mutuamente excluyente a recursos compartidos globales (como contadores de tareas `g_tasks_cnt`). Luego de modificar el estado, libera el mutex con `xSemaphoreGive()` y realiza la señalización necesaria para coordinarse con el resto del sistema.

* **task_exit_a.c**: Representa la contraparte lógica de la tarea de entrada, encargándose de la salida o liberación de la zona restringida. De forma similar, la tarea espera a que se active el evento de salida usando `xSemaphoreTake()`. Luego adquiere el mutex de control compartido para actualizar de forma segura el estado o contador de ocupación (por ejemplo, decrementando la ocupación). Según el estado resultante (si la zona queda libre o disponible), libera semáforos para avisar a otras tareas u operar compuertas, entregando finalmente el mutex.

* **task_test.c**: Es una tarea de simulación que tiene como propósito probar el sistema estimulándolo con eventos periódicos. Cuenta con un arreglo de variables enumeradas (por ejemplo `e_task_test_array`) que representan una secuencia de peticiones simuladas. La tarea recorre el arreglo cíclicamente y utiliza la función de retardo de precisión de FreeRTOS (`vTaskDelayUntil()`) para inyectar cada evento a un ritmo predefinido, disparando cada suceso mediante un `xSemaphoreGive()` hacia los semáforos que las demás tareas se encuentran aguardando.

* **freertos.c**: Archivo que implementa los *hooks* de la aplicación (funciones *callback* que el RTOS llama en escenarios previstos del kernel). Contiene `vApplicationIdleHook()` que se ejecuta cuando el sistema está inactivo porque no hay otra tarea lista para ejecutarse, `vApplicationTickHook()` que se dispara con cada interrupción del temporizador del sistema (Tick), y `vApplicationStackOverflowHook()`, que se ejecuta si se detecta un desborde en el límite de la pila de alguna tarea, interrumpiendo la ejecución (`configASSERT( 0 )`) para propósitos de depuración.

