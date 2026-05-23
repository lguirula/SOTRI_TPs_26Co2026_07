# CESE - Sistemas Operativos de Tiempo Real

## Trabajo Practico N°: 1 - Tareas de FreeRTOS - Actividad 4



### Paso 2
**¿Cómo implementar el procesamiento periódico mediante una Tarea?**

En FreeRTOS, el enfoque recomendado para una tarea periódica es usar `vTaskDelayUntil()`, ya que mantiene un período constante y evita deriva temporal acumulada.

La idea general es:

1. Definir el período en ticks (`pdMS_TO_TICKS(...)`).
2. Guardar una referencia temporal inicial con `xTaskGetTickCount()`.
3. Ejecutar el procesamiento en un bucle infinito.
4. Bloquear la tarea hasta el próximo instante periódico con `vTaskDelayUntil()`.

Ejemplo:

```c
void task_periodic(void *pvParameters)
{
    TickType_t xLastWakeTime = xTaskGetTickCount();
    const TickType_t xPeriod = pdMS_TO_TICKS(100);

    for (;;)
    {
        process_data();
        vTaskDelayUntil(&xLastWakeTime, xPeriod);
    }
}
```

Con este esquema, `process_data()` se ejecuta cada 100 ms (según el tick del sistema y el tiempo de ejecucion interno de la funcion).

---

**¿Cuándo se ejecutará la Tarea IDLE y cómo se puede utilizar?**

La tarea IDLE se ejecuta automáticamente cuando no hay ninguna otra tarea en estado `Ready` con prioridad mayor a `tskIDLE_PRIORITY`.

En otras palabras, corre cuando el CPU queda libre.

Usos típicos de la tarea IDLE:

1. **Liberación de recursos internos de FreeRTOS**  
   FreeRTOS utiliza la IDLE para liberar memoria de tareas eliminadas con `vTaskDelete()`.

2. **Hook de bajo consumo o mantenimiento liviano**  
   Si en `FreeRTOSConfig.h` está habilitado `configUSE_IDLE_HOOK`, se puede implementar:

```c
void vApplicationIdleHook(void)
{
    __WFI();
}
```

Esto permite ejecutar acciones de fondo muy cortas o instruir al microcontrolador para entrar en espera de interrupcion (`WFI`) y reducir consumo.

Buenas prácticas:

- No bloquear la IDLE (no usar demoras bloqueantes dentro del hook).
- No colocar procesamiento pesado en `vApplicationIdleHook()`.
- Priorizar tareas explícitas para lógica funcional, y dejar la IDLE para mantenimiento mínimo o ahorro energético.

### Paso 3
**Modificar `task_led` para implementar procesamiento periodico, compilar/depurar/observar.**

Al convertir `task_led` en una tarea periodica con `vTaskDelayUntil()`, y observar con la herramienta Live Expressions del IDE, se verifica que la tarea del LED se ejecuta con una frecuencia estable y menor que la tarea del boton, que continua con un delay relativo. Esto evidencia un mejor reparto del scheduler y, en consecuencia, un uso de CPU mas ordenado y predecible.

### Paso 4
**Modificar `task_btn` para implementar procesamiento periodico, compilar/depurar/observar.**

Al convertir tambien `task_btn` en una tarea periodica con `vTaskDelayUntil()`, y observar nuevamente con Live Expressions del IDE, se verifica que ambas tareas (`task_led` y `task_btn`) quedan sincronizadas a sus periodos configurados, con activaciones mas regulares y sin deriva acumulada ciclo a ciclo. Esto es notable comparado con la tarea de Idle. Como resultado, el comportamiento general del sistema se vuelve mas deterministico, el scheduler reparte mejor el tiempo de CPU y se facilita la depuracion temporal de eventos.

