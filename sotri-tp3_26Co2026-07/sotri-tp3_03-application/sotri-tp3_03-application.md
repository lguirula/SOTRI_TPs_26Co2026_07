# CESE - Sistemas Operativos de Tiempo Real

## Trabajo Practico N°: 3 - Sincronizacion de Tareas de FreeRTOS (Actividad 03)

> **Proyecto:** `sotri-tp3_03-application`  
> **Aplicacion:** `sotri-tp3_03-application: Vehicular crossing` (cruce vial de capacidad limitada)  
> **Placa de referencia en la guia:** NUCLEO-F103RB (STM32F446RE en este esqueleto ORIGINAL). La arquitectura de la capa APP es equivalente.

---

## Paso 03: Analisis del codigo fuente base

Analisis y explicacion (en espanol) del funcionamiento del codigo fuente contenido en los archivos: `app.c`, `app_it.c`, `task_entry_a.c`, `task_exit_a.c`, `task_test.c` y `freertos.c`, segun la guia TP3 Actividad 03 (Paso 03).

### Contexto del problema (Vehicular crossing)

Segun la guia TP3 Actividad 03, Paso 06, el sistema a implementar es un **control de acceso a un cruce vial** de capacidad limitada:

- Dos puntos de **ingreso** (`task_entry_a`, `task_entry_b`) y dos de **egreso** (`task_exit_a`, `task_exit_b`).
- Los vehiculos ingresan y egresan por orden de llegada hasta alcanzar la capacidad maxima **`G_TASKS_CNT_MAX`** (definido en `app.h` como `3ul`).
- Cada punto de ingreso tiene un **semaforo vial** (Rojo / Verde) a controlar.
- Una quinta tarea **`task_test`** inyecta estimulos de prueba desde un arreglo: `Entry_A`, `Exit_A`, `Entry_B`, `Exit_B` (en el codigo: `Entry_A`, `Exit_A`, etc. del enum `e_task_test_t`).

El codigo analizado aqui corresponde al **esqueleto inicial entregado** (antes del Paso 06): crea las tareas, reserva la estructura para primitivas RTOS y define el simulador `task_test`, pero **aun no** crea semaforos/mutex ni implementa la logica de cruce.

---

### 1. `app.c` – Orquestacion e inicializacion

**Rol:** nucleo de la capa de aplicacion. Se invoca desde `main.c` **antes** de `osKernelStart()`.

**Identificacion del sistema (lineas 66-68, `app.c`):**

- Describe la aplicacion como un sistema **orientado a eventos** (*Event-Triggered Systems*, ETS).
- Nombre del proyecto: *Vehicular crossing*.

**Flujo de `app_init()`:**

1. **Inicializa contadores globales** (`g_app_cnt`, `g_app_task_cnt`, `g_app_tick_cnt`, `g_task_idle_cnt`, `g_app_stack_overflow_cnt`, `g_tasks_cnt`) en cero.
2. **Log de identificacion** via `LOGGER_INFO`: nombre de la aplicacion y tick actual (`xTaskGetTickCount()`).
3. **Reserva de primitivas RTOS:** bloques comentados (lineas 79-83 y 112-118, `app.c`) para colas, semaforos binarios y mutex. En el esqueleto **no se invoca aun** `xSemaphoreCreateBinary()` ni `xSemaphoreCreateMutex()`. Los handles estan previstos en `app.h` (`h_entry_a_bin_sem`, `h_exit_a_bin_sem`, `h_continue_bin_sem`, `h_mutex_mut_sem`) pero sin instancia creada.
4. **Creacion de cinco tareas** con `xTaskCreate` y verificacion `configASSERT(pdPASS == ret)`:

| Tarea | Funcion | Prioridad (`tskIDLE_PRIORITY + N`) | Rol conceptual |
|-------|---------|-------------------------------------|----------------|
| Task Entry A | `task_entry_a` | **+3** | Ingreso de vehiculos – acceso A |
| Task Exit A | `task_exit_a` | **+2** | Egreso de vehiculos – salida A |
| Task Entry B | `task_entry_b` | **+3** | Ingreso de vehiculos – acceso B |
| Task Exit B | `task_exit_b` | **+2** | Egreso de vehiculos – salida B |
| Task Test | `task_test` | **+1** | Simulador automatico de eventos |

**Importante sobre prioridades:** los comentarios en el codigo dicen "This task will run at priority 1" pero los valores reales son +3, +2 y +1. Las tareas de ingreso (Entry A/B) tienen la **mayor** prioridad operativa; las de egreso (Exit A/B) prioridad intermedia; el test arranca en la **mas baja**.

5. **Consulta heap libre** con `xPortGetFreeHeapSize()` (diagnostico de memoria dinamica del kernel).
6. **`app_it_init()`** – inicializacion de interrupciones de aplicacion.
7. **`cycle_counter_init()`** – contador DWT para medicion de tiempos en depuracion.

**Observacion del esqueleto:** `app.c` incluye headers de `task_entry_b` y `task_exit_b` y crea esas tareas, aunque el Paso 03 de la guia pide analizar principalmente `task_entry_a.c` y `task_exit_a.c`. Las tareas B son **simetricas** en estructura (mismo patron de `vTaskDelay` de 2500 ms) y se sincronizaran en el Paso 06 siguiendo el mismo esquema que A.

---

### 2. `app_it.c` – Interrupciones de aplicacion

**Rol:** capa entre el hardware (boton EXTI) y las tareas de aplicacion. En un sistema ETS, las interrupciones son una fuente alternativa de estimulos ademas de `task_test`.

**`app_it_init()` (lineas 57-65, `app_it.c`):**

- Plantilla con `CPSID i` / `CPSIE i` para una seccion critica muy corta al deshabilitar y rehabilitar interrupciones globales.
- Hoy no configura NVIC ni EXTI adicional; solo reserva el lugar para inicializacion futura.

**`HAL_GPIO_EXTI_Callback(uint16_t GPIO_Pin)` (lineas 72-79, `app_it.c`):**

- Callback de la capa HAL de STM32: se ejecuta cuando un pin configurado como interrupcion externa (EXTI) detecta el flanco configurado.
- Verifica si el pin es `BTN_A_PIN`.
- El cuerpo esta **vacio** (`/* Work to be done. */`): pendiente para el Paso 06 (tipicamente `xSemaphoreGiveFromISR()` hacia un semaforo binario + `portYIELD_FROM_ISR()`).

En la solucion final del cruce vial, el boton fisico puede complementar a `task_test` como fuente de eventos `Entry_*` / `Exit_*`.

---

### 3. `task_entry_a.c` – Monitoreo y control del ingreso (acceso A)

**Rol segun la guia:** `task_entry_a` monitorea y controla el **ingreso** de vehiculos por el punto A.

**Estado actual (esqueleto):**

```text
task_entry_a()
  ├─ Inicializa g_task_entry_a_cnt = 0
  ├─ LOGGER_INFO: arranque con nombre de tarea y tick
  └─ for (;;)
       ├─ g_task_entry_a_cnt++
       ├─ log "Wait 2500mS"
       └─ vTaskDelay(TASK_ENTRY_A_DEL_MAX)   // 2500 ms
```

- **No** espera semaforos (`xSemaphoreTake`) ni mutex.
- **No** controla semaforo vial (Rojo/Verde).
- **No** actualiza `g_tasks_cnt` ni verifica `G_TASKS_CNT_MAX`.
- Solo demuestra que la tarea existe, corre y cede la CPU cada 2,5 s mediante `vTaskDelay()`.

**Comportamiento esperado tras Paso 06 (guia):**

- Bloqueo en `h_entry_a_bin_sem` al recibir estimulo `Entry_A` (desde `task_test` o ISR).
- Toma de `h_mutex_mut_sem` para acceder de forma exclusiva al contador compartido `g_tasks_cnt`.
- Si `g_tasks_cnt == G_TASKS_CNT_MAX`, impedir nuevo ingreso y esperar en `h_continue_bin_sem`.
- Control del semaforo vial A (Rojo/Verde) mediante semaforo binario dedicado (`h_traffic_light_a_bin_sem` en la solucion completa).
- Coordinacion con el acceso B cuando el carril se llena o se libera.

**Comparacion con baremetal:** en lugar de un bucle que hace `delay()` y consulta flags, aqui la tarea **se bloquea** en primitivas RTOS hasta que llega un evento; el scheduler ejecuta otras tareas mientras espera.

---

### 4. `task_exit_a.c` – Monitoreo y control del egreso (salida A)

**Rol segun la guia:** `task_exit_a` monitorea y controla el **egreso** de vehiculos por el punto A.

**Estado actual (esqueleto):** estructura **identica** a `task_entry_a.c`:

```text
task_exit_a()
  ├─ Inicializa g_task_exit_a_cnt = 0
  ├─ LOGGER_INFO: arranque
  └─ for (;;)
       ├─ g_task_exit_a_cnt++
       ├─ log "Wait 2500mS"
       └─ vTaskDelay(TASK_EXIT_A_DEL_MAX)   // 2500 ms
```

- **No** espera `h_exit_a_bin_sem`.
- **No** decrementa `g_tasks_cnt` bajo mutex.
- **No** libera semaforo vial ni senaliza `h_continue_bin_sem` cuando el cruce deja de estar lleno.

**Comportamiento esperado tras Paso 06:**

- Bloqueo en `h_exit_a_bin_sem` ante estimulo `Exit_A`.
- Toma de mutex, decremento de `g_tasks_cnt`.
- Si el cruce pasa de lleno a `(G_TASKS_CNT_MAX - 1)`, dar `xSemaphoreGive(h_continue_bin_sem)` para desbloquear ingresos.
- Si `g_tasks_cnt == 0`, liberar semaforo vial B (vehiculos del acceso B pueden ingresar).

---

### 5. `task_test.c` – Simulador automatizado de eventos

**Rol:** tarea generadora de escenarios que inyecta estimulos a las tareas Entry/Exit **sin depender del boton fisico**. Es clave en la metodologia del curso para probar sincronizacion de forma repetible.

**Enum de eventos (lineas 56, `task_test.c`):**

```c
typedef enum e_task_test {Error, Entry_A, Entry_B, Exit_A, Exit_B} e_task_test_t;
```

Corresponde a los estimulos de la guia: `ENTRY_A`, `EXIT_A`, `ENTRY_B`, `EXIT_B`.

**Escenarios configurables (`E_TASK_TEST_X`, lineas 72-102, `task_test.c`):**

| Valor | Contenido del array `e_task_test_array[]` | Proposito |
|-------|---------------------------------------------|-----------|
| 0 | `{Error, Exit_B+1, Exit_B+2}` | Valores invalidos → rama `default` |
| 1 | `{Entry_A, Exit_A}` | Escenario minimo acceso A (activo con `#define E_TASK_TEST_X (1)`) |
| 2 | `{Entry_A, Entry_A, Exit_A, Exit_A}` | Dos ingresos y dos egresos A |
| 3 | `{Entry_A, Entry_A, Entry_A, Exit_A, Exit_A, Exit_A}` | Llenar capacidad y vaciar |
| 4 | Ocho eventos alternados | Secuencia extendida |
| 5 | Diez eventos alternados | Secuencia extendida |

**Secuencia de arranque (lineas 114-138, `task_test.c`):**

1. `last_wake_time = xTaskGetTickCount()` para usar `vTaskDelayUntil` (periodo estable).
2. Lee prioridad actual: `uxTaskPriorityGet(NULL) + 2ul`.
3. **`vTaskPrioritySet(NULL, task_test_priority)`** → sube de prioridad **1** a **3**, igual que Entry A/B.
4. Motivo (comentario en codigo): garantizar que el banco de pruebas se ejecute sin ser preemptada por tareas de prioridad 2 (Exit) durante la fase de configuracion inicial del escenario.

**Bucle principal (lineas 141-177, `task_test.c`):**

- Bucle externo infinito; bucle interno recorre `e_task_test_array[]`.
- Por cada elemento: incrementa `g_task_test_cnt`, loguea indice, ejecuta `switch`.
- **`case Entry_A` y `case Exit_A`:** cuerpo **vacio** (solo `break`) — no hay aun `xSemaphoreGive(h_entry_a_bin_sem)` ni `xSemaphoreGive(h_exit_a_bin_sem)`.
- **`case Error` / `default`:** loguea mensaje de error.
- Tras cada evento: `vTaskDelayUntil(&last_wake_time, 5000 ms)` → un estimulo cada **5 segundos**, periodico y estable.

**Comportamiento esperado tras Paso 06:**

- `Entry_A` → `xSemaphoreGive(h_entry_a_bin_sem)` (desbloquea `task_entry_a`).
- `Exit_A` → `xSemaphoreGive(h_exit_a_bin_sem)` (desbloquea `task_exit_a`).
- Idem para B con `h_entry_b_bin_sem` / `h_exit_b_bin_sem`.

---

### 6. `freertos.c` – Hooks del kernel (capa APP)

**Rol:** implementa callbacks opcionales habilitados en `FreeRTOSConfig.h`. **No participan** en la logica del cruce vial; sirven para diagnostico y estadisticas del curso.

| Hook | Funcion | Comportamiento (lineas en `app/src/freertos.c`) |
|------|---------|---------------------------------------------------|
| `vApplicationIdleHook` | Tarea Idle | Incrementa `g_task_idle_cnt` (75) cada vez que no hay tareas Ready de mayor prioridad. |
| `vApplicationTickHook` | Cada tick del SO (~1 ms) | Incrementa `g_app_tick_cnt` (88). Ejecuta en contexto ISR: debe ser muy breve. |
| `vApplicationStackOverflowHook` | Desbordamiento de pila | `taskENTER_CRITICAL()`, `configASSERT(0)` para detener en depuracion (98-99), incrementa `g_app_stack_overflow_cnt` (103). |

**Nota:** existe otro `freertos.c` en `Core/Src/` generado por CubeMX con prototipos `__weak` vacios; la implementacion activa con logica de contadores esta en `app/src/freertos.c`.

---

### Diagrama de arquitectura (estado actual – esqueleto)

```text
main()
  └─ app_init()
       ├─ (pendiente) xSemaphoreCreateBinary / xSemaphoreCreateMutex
       ├─ xTaskCreate(task_entry_a, P=3)
       ├─ xTaskCreate(task_exit_a,  P=2)
       ├─ xTaskCreate(task_entry_b, P=3)
       ├─ xTaskCreate(task_exit_b,  P=2)
       ├─ xTaskCreate(task_test,    P=1 → luego P=3)
       ├─ app_it_init()
       └─ cycle_counter_init()
  └─ osKernelStart()
       │
       ├─ task_entry_a ── vTaskDelay(2500ms) ── (sin semaforos)
       ├─ task_exit_a  ── vTaskDelay(2500ms) ── (sin semaforos)
       ├─ task_entry_b ── vTaskDelay(2500ms) ── (sin semaforos)
       ├─ task_exit_b  ── vTaskDelay(2500ms) ── (sin semaforos)
       ├─ task_test    ── recorre array ── switch vacio ── vTaskDelayUntil(5000ms)
       ├─ BTN_A (EXTI) ── HAL_GPIO_EXTI_Callback ── (vacio)
       └─ Idle / Tick hooks ── contadores g_task_idle_cnt, g_app_tick_cnt
```

---

### Orden de ejecucion inicial (prioridades verificadas)

Al arrancar el scheduler, las tareas Entry A/B (P=3) pueden ejecutar primero. `task_test` arranca con P=1 pero **inmediatamente** se eleva a P=3 y loguea su prioridad antes de entrar al bucle de eventos. Las tareas Exit (P=2) quedan entre el test inicial y las Entry en terminos de preemptacion.

En el esqueleto actual, todas las tareas operativas terminan rapidamente en `vTaskDelay()`, cediendo CPU; no hay contencion real ni sincronizacion entre ellas.

---

### Resumen: esqueleto vs Paso 06 (guia)

| Elemento | Estado en esqueleto ORIGINAL | Paso 06 (implementacion) |
|----------|------------------------------|---------------------------|
| Semaforos binarios Entry/Exit | Declarados en `app.h`, no creados en `app.c` | `xSemaphoreCreateBinary` + Give/Take |
| Mutex `g_tasks_cnt` | No usado | Proteger incremento/decremento |
| Semaforo vial (Rojo/Verde) | No presente | Control de acceso por acceso A/B |
| `h_continue_bin_sem` | Declarado, no usado | Desbloquear ingresos al liberar capacidad |
| `task_test` switch | Cases vacios | `xSemaphoreGive` por evento |
| `HAL_GPIO_EXTI_Callback` | Vacio | Give desde ISR |
| Capacidad `G_TASKS_CNT_MAX` | Definido (`3ul`), no enforced | Contador + mutex + continue |

---

### Referencia guia TP3 – Actividad 03

- **Paso 03:** analizar y explicar el codigo fuente base (este documento).
- **Paso 04:** depurar y confirmar el comportamiento descrito.
- **Paso 06:** implementar sincronizacion con semaforos binarios y mutex para el problema *Vehicular crossing*, siguiendo los pasos recomendados de la guia (Entry/Exit A, luego B, mutex compartido, semaforo vial, limite de capacidad).
