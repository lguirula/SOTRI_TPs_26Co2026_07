# CESE - Sistemas Operativos de Tiempo Real

## Trabajo Practico N°: 1 - Tareas de FreeRTOS - Actividad 3



### Paso 2
**¿Cómo usar el parámetro de Tarea?** 

Al momento de crear una tarea con `xTaskCreate()`, el cuarto argumento (`void * const pvParameters`) nos permite enviarle información inicial a la función de la tarea (Task Function). 

Este parámetro es del tipo puntero a *void* (`void *`), lo que lo hace totalmente genérico: podemos pasarle la dirección de memoria de un número entero, de un arreglo, de una estructura completa de configuración (`struct`) que contenga múltiples datos, **o incluso punteros a funciones** (callbacks).

```c
// Ejemplo: pasando un valor estático por referencia
static uint32_t config_value = 1;

ret = xTaskCreate(my_task, 
                  "My Task", 
                  STACK_SIZE, 
                  (void *)&config_value, /* <--- PASAMOS EL PARÁMETRO AQUÍ */
                  PRIORITY, 
                  &handle);
```

Luego, dentro de la **Task Function**, lo primero que se debe hacer es castear (convertir) ese puntero genérico de vuelta al tipo de dato original para poder utilizarlo:

```c
void my_task(void *parameters)
{
    // 1. Casteamos el puntero genérico al tipo esperado y leemos su contenido
    uint32_t my_config = *((uint32_t *)parameters);
    
    // 2. Usamos el parámetro para decidir cómo inicializar la tarea
    if (my_config == 1) {
        // Comportamiento A
    } else {
        // Comportamiento B
    }

    // Bucle infinito de la tarea
    for(;;) {
        // ...
    }
}
```

Gracias a este parámetro genérico, podemos hacer que el código de una tarea sea reutilizable. Es decir, podemos usar la misma **Task Function** para crear múltiples instancias que se comporten de forma distinta según los parámetros de configuración que recibieron al nacer.




**¿Cómo cambiar la prioridad de una Tarea ya creada?**

Para cambiar la prioridad de una tarea en tiempo de ejecución (una vez que ya fue creada/iniciada), se utiliza la función de la API de FreeRTOS `vTaskPrioritySet()`.

**Prototipo:**
```c
void vTaskPrioritySet( TaskHandle_t xTask, UBaseType_t uxNewPriority );
```

**Cambiar la prioridad de OTRA tarea:**
```c
// Si queremos elevar la prioridad de task_btn a 3:
vTaskPrioritySet(h_task_led, (tskIDLE_PRIORITY + 3ul));
```

**Cambiar la prioridad de LA MISMA tarea que está ejecutando:**
```c
// La tarea actual reduce su propia prioridad a 1:
vTaskPrioritySet(NULL, (tskIDLE_PRIORITY + 1ul));
```

En el código la creamos con prioridad más alta, y una vez que se ejecuta la *Task Function*, antes del `while(1)`, volvemos la prioridad a su valor normal con:
```c
vTaskPrioritySet(NULL, (tskIDLE_PRIORITY + 1ul));
```



### Paso 3

En este paso, se aprovecha el parámetro `void * const pvParameters` de `xTaskCreate()` para pasar información de configuración a cada instancia de `task_btn`.

La creación de la tarea queda de la siguiente manera:

```c

int button_1 = 1;
int button_2 = 2;

    /* Task BTN thread at priority 1 */
    ret = xTaskCreate(task_btn,							/* Pointer to the function thats implement the task */
                    "Task BTN 1",						/* Text name for the task. This is to facilitate debugging only. */
                    (2 * configMINIMAL_STACK_SIZE),	/* Stack depth in words. */
                    (void *) button_1,				/* We are not using the task parameter. */
                    (tskIDLE_PRIORITY + 1ul),			/* This task will run at priority 1. */
                    &h_task_btn_1);						/* We are using a variable as task handle. */

```

En lugar de enviar `NULL`, se pasa un puntero a un entero que identifica el boton a utilizar (esta decodificacion se realiza con un `switch` dentro de la tarea).

Para controlar dos botones distintos, se definieron dos estructuras de configuración:

```c
task_btn_dta_t task_btn_dta_1 = {
    EV_BTN_UP, ST_BTN_UP, DEL_BTN_MIN,
    B1_GPIO_Port, B1_Pin
};

task_btn_dta_t task_btn_dta_2 = {
    EV_BTN_UP, ST_BTN_UP, DEL_BTN_MIN,
    B2_GPIO_Port, B2_Pin
};
```

Cada estructura apunta a un GPIO diferente. Luego, dentro de `task_btn`, se castea `pvParameters` a `task_btn_dta_t *` para que cada instancia opere sobre su botón correspondiente.

```c
int button = parameters != NULL ? (int)parameters : 0;

	switch(button) {
	case 1:
		task_btn_dta = &task_btn_dta_1;
		break;
	case 2:
	default:
		task_btn_dta = &task_btn_dta_2;
		break;
	}
```


Logs:
[info]  
[info] app_init is running - Tick [mS] = 0
[info]  app is a RTOS - Event-Triggered Systems (ETS)
[info]  app is a seo-tp1_01-application: Demo Code
[info]  app is a (Source => CESE - Sistemas Operativos de Tiempo Real
[info]  
[info] Task LED is running - Tick [mS] =   0
[info]  
[info] Task BTN 1 is running - Tick [mS] =   0
[info]  
[info] Task BTN 2 is running - Tick [mS] =   1
[info]  Task BTN 1 - BTN PRESSED
[info]  Task BTN 2 - BTN PRESSED
[info]  Task LED - LED BLINK
[info]  Task BTN 1 - BTN HOVER
[info]  Task LED - LED OFF
[info]  Task BTN 1 - BTN PRESSED
[info]  Task LED - LED BLINK
[info]  Task BTN 2 - BTN HOVER
[info]  Task LED - LED OFF
[info]  Task BTN 2 - BTN PRESSED
[info]  Task LED - LED BLINK
[info]  Task BTN 2 - BTN HOVER



### Paso 4: Modificar la prioridad de task_led al inicio

El objetivo de este paso es garantizar que `task_led` sea **estrictamente la primera tarea en ejecutarse** cuando el scheduler de FreeRTOS arranca, independientemente de qué otras tareas existan. Para lograrlo y luego restablecer el comportamiento normal (donde `task_btn` y `task_led` comparten la CPU o tienen prioridades relativas específicas), se aplica la siguiente técnica en dos fases:

**1. Creación con prioridad inflada:**
Al momento de crear la tarea en `app_init` usando `xTaskCreate()`, le asignamos temporalmente una prioridad artificialmente alta (por ejemplo, `tskIDLE_PRIORITY + 3ul`). Esto asegura que cuando se llame a `osKernelStart()`, el Scheduler vea que `task_led` es la tarea más prioritaria de todo el sistema y le entregue el control de la CPU inmediatamente.

```c
ret = xTaskCreate(task_led, "Task LED", STACK_SIZE, NULL, 
                  (tskIDLE_PRIORITY + 3ul), // <--- Prioridad Alta
                  &h_task_led);
```

**2. Restablecimiento de la prioridad dentro de la tarea:**
Apenas arranca a ejecutarse la *Task Function* (`task_led`), la tarea realiza sus rutinas de inicialización temprana. Justo antes de entrar a su bucle infinito `for(;;)` (o `while(1)`), la tarea usa la API `vTaskPrioritySet()` para bajar su propia prioridad de vuelta a su valor original de diseño (por ejemplo, `tskIDLE_PRIORITY + 1ul`).

```c
void task_led(void *parameters)
{
    // Inicializaciones ...
    LOGGER_INFO("Task LED is running...");

    // La tarea se baja la prioridad a sí misma al valor normal de operación
    vTaskPrioritySet(NULL, (tskIDLE_PRIORITY + 1ul));

    // Bucle infinito normal
    for(;;)
    {
        task_led_statechart();
    }
}
```

