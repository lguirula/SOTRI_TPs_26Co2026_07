# CESE - Sistemas Operativos de Tiempo Real

## Trabajo Practico N°: 3 - Sincronizacion de Tareas de FreeRTOS (Actividad 04)

> **Proyecto:** `sotri-tp3_04-application`  
> **Aplicacion:** `sotri-tp3_04-application: Security airlock` (puerta esclusa de seguridad)  
> **Placa:** NUCLEO-L4R5ZI (STM32L4R5ZI). La guia original referencia NUCLEO-F103RB; la arquitectura APP es equivalente.


### Paso 03 :

El codigo fuente de este Trabajo Practico establece una arquitectura basada en **FreeRTOS** orientada a eventos, disenada para modelar una **puerta esclusa de seguridad** (*Security airlock*). A diferencia de la Actividad 03 (*Vehicular crossing*), aqui las tareas operativas son cuatro **compuertas** (`task_gate_a` … `task_gate_d`) mas una tarea de prueba (`task_test`) que inyecta estimulos simulados. El esqueleto entregado crea las tareas y el simulador, pero **aun no implementa** semaforos, mutex ni la logica de enclavamiento entre puertas (eso corresponde al Paso 06 de la guia).

### Nota sobre nombres de archivos (guia vs proyecto)

La guia TP3 Actividad 04, Paso 03, cita `task_entry_a.c` y `task_exit_a.c` (heredado de la Actividad 03). En **este** proyecto las tareas equivalentes son:

| Guia (texto generico) | Archivo real en TP3-04 |
|-----------------------|-------------------------|
| `task_entry_a.c` / `task_exit_a.c` | `task_gate_a.c` … `task_gate_d.c` |

Cada `task_gate_*` modela un punto de ingreso/egreso de la esclusa. El analisis de `task_gate_a.c` aplica a B, C y D (misma estructura en el esqueleto entregado).

### Contexto del problema (Security airlock)

Segun la guia TP3, el problema *Security airlock* describe un sistema donde:

- Varias personas pueden solicitar paso por **cuatro puntos** (puertas A, B, C, D).
- Solo **una puerta abierta a la vez**; normalmente todas **cerradas**.
- Estimulos de prueba: `OPEN_REQUEST_*` (pedido de apertura) y `DOOR_CLOSED_*` (puerta cerrada).
- Sincronizacion con **semaforos binarios** y **mutex** para recursos compartidos.

El codigo analizado aqui corresponde al **esqueleto inicial**: crea tareas, interrupciones y un simulador, pero **aun no** implementa mutex ni logica de esclusa.

---

### 1. `app.c` (Configuracion e Inicializacion Principal)

Actua como el nucleo orquestador de la capa de aplicacion. Se invoca desde `main.c` **antes** de `osKernelStart()`.

**Flujo de `app_init()`:**

1. **Inicializa contadores globales** (`g_app_cnt`, `g_app_task_cnt`, `g_app_tick_cnt`, etc.) en cero.
2. **Log de identificacion** via `LOGGER_INFO`: identifica la aplicacion como *Security airlock* y muestra el tick actual.
3. **Reserva de primitivas RTOS:** bloques comentados para colas, semaforos y mutex. En el esqueleto **no se crean aun** (`xSemaphoreCreateBinary`, `xSemaphoreCreateMutex`, etc. pendientes para Paso 06).
4. **Creacion de cinco tareas** con `xTaskCreate` y `configASSERT(pdPASS == ret)`:

| Tarea | Funcion | Prioridad real | Rol conceptual |
|-------|---------|----------------|----------------|
| Task Gate A | `task_gate_a` | `tskIDLE_PRIORITY + 3` | Puerta A – ingreso/egreso |
| Task Gate B | `task_gate_b` | `tskIDLE_PRIORITY + 2` | Puerta B |
| Task Gate C | `task_gate_c` | `tskIDLE_PRIORITY + 3` | Puerta C |
| Task Gate D | `task_gate_d` | `tskIDLE_PRIORITY + 2` | Puerta D |
| Task Test | `task_test` | `tskIDLE_PRIORITY + 1` | Simulador de eventos |

**Importante:** las puertas **no** arrancan con la misma prioridad. A/C tienen prioridad **3**, B/D prioridad **2**, y el test arranca en **1** (la mas baja). Los comentarios en el codigo dicen "priority 1" pero los valores numericos son +3, +2 y +1.

5. **Consulta heap libre** con `xPortGetFreeHeapSize()` (diagnostico).
6. **`app_it_init()`** – inicializacion de interrupciones de aplicacion.
7. **`cycle_counter_init()`** – contador DWT para medicion de tiempos.

**Observaciones del esqueleto:**

- `app.h` aun declara handles de la Actividad 03 (`h_entry_a_bin_sem`, `task_entry_a`, etc.) que no coinciden con `app.c`.
- En `xTaskCreate` de Gate C, el nombre de depuracion dice `"Task Gate B"` en lugar de `"Task Gate C"` (linea 147, `app.c`).

---

### 2. `app_it.c` (Manejo de Interrupciones)

Este archivo conecta el hardware (boton EXTI) con las tareas de aplicacion.

* **`app_it_init()`:** plantilla con `CPSID i` / `CPSIE i` para una seccion critica corta al deshabilitar/habilitar interrupciones. Hoy no configura NVIC ni EXTI adicional.
* **`HAL_GPIO_EXTI_Callback(uint16_t GPIO_Pin)`:** callback nativo de la capa HAL de STM32 que se dispara cuando un pin configurado como interrupcion cambia de estado. Detecta `BTN_A_PIN` pero el cuerpo esta **vacio** (TODO Paso 06: delegar a tarea con `xSemaphoreGiveFromISR` + `portYIELD_FROM_ISR`).

En la esclusa final, el boton fisico puede complementar a `task_test` como fuente alternativa de `OPEN_REQUEST_*`.

---

### 3. `task_gate_a.c`, `task_gate_b.c`, `task_gate_c.c`, `task_gate_d.c` (Logica de las Compuertas)

Cada archivo modela el comportamiento de una compuerta. Tienen estructura arquitectonica identica; sustituyen a `task_entry_a` / `task_exit_a` de la Actividad 03.

**Estado actual (esqueleto):**

```text
task_gate_x()
  ├─ Inicializa g_task_gate_x_cnt
  ├─ LOGGER_INFO: arranque con nombre de tarea y tick
  └─ for (;;)
       ├─ g_task_gate_x_cnt++
       ├─ log "Wait 2500mS"
       └─ vTaskDelay(2500 ms)
```

* **Inicializacion:** cada tarea declara e inicializa un contador individual (`g_task_gate_a_cnt`, `g_task_gate_b_cnt`, etc.) y reporta por consola (`LOGGER_INFO`) que ha comenzado a correr junto con la marca de tiempo (ticks) actual.
* **Ciclo de trabajo:** ingresan a un bucle infinito con demora maxima de 2500 milisegundos (`TASK_GATE_X_DEL_MAX`). Solo demuestran que la tarea existe, corre y cede CPU cada 2,5 s.
* **No** esperan semaforos ni mutex.
* **No** controlan LED/puerta (Open/Close).

**Comportamiento esperado tras Paso 06 (guia):**

- Bloqueo en semaforo binario al recibir `OPEN_REQUEST_X` (desde `task_test` o ISR).
- Mutex compartido entre puertas que compiten por "abrir una sola a la vez".
- Accion Open/Close sobre el actuador (LED en la placa).
- Señal `DOOR_CLOSED_X` cuando la puerta vuelve a cerrarse.

**Detalle menor:** en `task_gate_d.c` (linea 59), el mensaje de log dice `"Task Gate A"` en lugar de `"Task Gate D"`.

---

### 4. `task_test.c` (Simulador o Maquina de Pruebas Automatizadas)

Es el nucleo de validacion logica del programa. Su objetivo es inyectar solicitudes programadas para observar como reaccionan y sincronizan las compuertas, sin depender del boton fisico.

**Enum de eventos:**

```c
typedef enum e_task_test {
    Error,
    OPEN_REQUEST_A, DOOR_CLOSED_A,
    OPEN_REQUEST_B, DOOR_CLOSED_B,
    OPEN_REQUEST_C, DOOR_CLOSED_C,
    OPEN_REQUEST_D, DOOR_CLOSED_D
} e_task_test_t;
```

**Escenarios (`E_TASK_TEST_X`):**

| Valor | Secuencia del array |
|-------|---------------------|
| 0 | Error + valores invalidos (prueba de default) |
| 1 | A: OPEN → CLOSED → OPEN → CLOSED |
| 2 | A completo, luego B |
| 3 | B, luego C |
| 4 | C, luego D |
| 5 | D, luego A (ciclo) |
| 6 | Secuencia completa A→B→C→D |

Con `#define E_TASK_TEST_X (1)` se usa el escenario **solo puerta A**.

**Secuencia de arranque:**

1. `last_wake_time = xTaskGetTickCount()` para `vTaskDelayUntil`.
2. Sube su prioridad con `vTaskPrioritySet(NULL, uxTaskPriorityGet(NULL) + 2ul)` → pasa de prioridad **1** a **3**, igual que Gate A/C, para ejecutar el banco de pruebas sin ser preemptada por las puertas de prioridad 2.
3. Bucle externo infinito; bucle interno recorre `e_task_test_array[]`.

**Switch de eventos:** los `case OPEN_REQUEST_*` y `DOOR_CLOSED_*` tienen cuerpo **vacio** (solo `break`). No hay aun `xSemaphoreGive` hacia `task_gate_*`. El `case Error` loguea mensaje de error. Notar que faltan los `case` para `OPEN_REQUEST_B` y `DOOR_CLOSED_B` en el switch (lineas 160-193, `task_test.c`).

**Temporizacion:** tras cada evento, `vTaskDelayUntil(..., 5000 ms)` → un estimulo cada 5 s, periodico.

**Contraste con Actividad 03:** alli `task_test` enviaba `xSemaphoreGive(h_entry_a_bin_sem)` etc. Aqui los estimulos tienen **dos fases por puerta** (apertura solicitada + puerta cerrada), acorde al problema *Security airlock*.

---

### Diagrama de arquitectura (estado actual – esqueleto)

```text
main()
  └─ app_init()
       ├─ xTaskCreate(task_gate_a..d, task_test)
       ├─ app_it_init()
       └─ cycle_counter_init()
  └─ osKernelStart()
       │
       ├─ task_gate_a..d  ──► vTaskDelay(2500 ms)  [sin sync aun]
       │
       ├─ task_test       ──► recorre array eventos ──► vTaskDelayUntil(5000 ms)
       │                      [switch vacio, sin Give/Take]
       │
       └─ HAL_GPIO_EXTI   ──► callback vacio [sin GiveFromISR]
```

---

### Orden de ejecucion esperado (validacion de prioridades)

En un MCU de un solo nucleo, solo **una** tarea ejecuta a la vez. Con las prioridades del esqueleto:

1. Tras `osKernelStart()`, las tareas Gate A/C (P=3) pueden ejecutar primero.
2. `task_test` (P=1) arranca, loguea, y **eleva su prioridad a 3** → compite con Gate A/C.
3. Cuando una tarea hace `vTaskDelay`, cede CPU y otra de igual o mayor prioridad puede correr.
4. Gate B/D (P=2) ejecutan cuando las de P=3 estan bloqueadas en delay.
5. **Sin sincronizacion cruzada** visible: las puertas y el test operan de forma independiente hasta completar Paso 06.

**Comportamiento esperado al depurar (Paso 04 – esqueleto):**

1. Logs de `app_init` con titulo *Security airlock*.
2. Arranque de **Task Gate A/B/C/D** y **Task Test**; `task_test` eleva prioridad y loguea su escenario.
3. **Task Test:** cada 5 s imprime indice del array y `Wait 5000mS`; con `E_TASK_TEST_X==1` alterna indices 0..3 (OPEN/CLOSED de A).
4. **Task Gate A..D:** cada ~2,5 s imprimen `Wait 2500mS` de forma independiente.

---

### Comparacion con baremetal

En baremetal, cuatro "puertas" y un generador de pruebas se implementarian con un **super-loop** y flags volatiles seteados por ISR del boton; la logica de "solo una abierta" seria un `if` global protegido con `__disable_irq()`. Con FreeRTOS, cada puerta es una **tarea independiente** que puede bloquearse en semaforos (`xSemaphoreTake`) sin consumir CPU, y el mutex garantiza exclusion mutua sin deshabilitar interrupciones. La tarea `task_test` reemplaza un timer software o una secuencia hardcodeada en el loop principal, permitiendo escenarios configurables via `E_TASK_TEST_X`.

---

### Referencia guia – Paso 06 (implementacion pendiente)

Objetivo: mutex + semaforos para que **solo una puerta este abierta**, con estimulos `OPEN_REQUEST_*` / `DOOR_CLOSED_*`.

Pasos recomendados:

1. **`task_test` + `task_gate_a`:** sincronizar pedido de apertura y puerta cerrada (semaforos binarios).
2. Repetir para `task_gate_b`, `task_gate_c`, `task_gate_d`.
3. **Mutex** entre tareas que comparten recurso "esclusa" (solo una abierta).
4. **Control Open/Close** (LED / logica de puerta).
5. Probar escenarios cambiando `E_TASK_TEST_X` (0..6).
