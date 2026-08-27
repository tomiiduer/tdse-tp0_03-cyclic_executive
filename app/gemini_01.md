# Análisis del Código Fuente y Evolución de Variables

## 1. Análisis del Funcionamiento del Código Fuente

El sistema implementa un **Ejecutivo Cíclico (*Cyclic Executive*)** sobre una arquitectura *Bare Metal*, conducido por eventos de tiempo mediante el temporizador **SysTick** configurado con un período base de **1 ms**.

### Descripción de los Archivos

* **`main.c` / `stm32f1xx_it.c` (Entorno e Interrupciones):**
  Inicializan el hardware del microcontrolador STM32, el contador de ciclos DWT (*Data Watchpoint and Trace*) utilizado para mediciones de tiempo precisas en microsegundos, y la interrupción periódica del SysTick.

* **`app_it.c` (Manejo de Ticks de Interrupción):**
  Define la variable global `g_app_tick_cnt`. Cada vez que el temporizador SysTick genera una interrupción (cada 1 ms), se ejecuta la función de respuesta `HAL_SYSTICK_Callback()`, la cual incrementa dicha variable.

* **`app.c` (Planificador y Loop Principal):**
  Contiene la lógica central de la aplicación a través de dos funciones principales:
  * `app_init()`: Inicializa la medición de ciclos DWT, ejecuta las funciones de inicialización de cada tarea (`task_a_init` y `task_b_init`), pone a cero las métricas de rendimiento (`NOE`, `LET`, `BCET`, `WCET`) e inicializa la variable `g_app_tick_cnt`.
  * `app_update()`: Implementa el bucle principal. En cada iteración extrae de manera atómica (deshabilitando temporalmente las interrupciones) los ticks acumulados en `g_app_tick_cnt`. Si existen ticks pendientes, ejecuta linealmente las tareas registradas (`task_a_update` y `task_b_update`), midiendo el tiempo de ejecución (`LET`) de cada una mediante el DWT. Al finalizar el procesamiento, pone la CPU en modo de bajo consumo mediante `HAL_PWR_EnterSLEEPMode` (`WFI`) hasta la llegada de la siguiente interrupción.

* **`task_a.c` (Tarea A):**
  Representa una tarea de procesamiento bloqueante.
  * En **`TEST_0`**: Ejecuta un bucle `for` iterativo de 1000 ciclos.
  * En **`TEST_1`**: Realiza una espera activa bloqueante de 20 ms mediante `HAL_Delay(20)`.

* **`task_b.c` (Tarea B):**
  Representa una tarea de procesamiento no bloqueante.
  * En **`TEST_0`**: Incrementa un contador interno de 0 a 50 de forma no bloqueante.
  * En **`TEST_1`**: Verifica el tiempo transcurrido mediante `HAL_GetTick()` sin bloquear el flujo de ejecución.

---

## 2. Evolución de las Variables del Sistema

### `g_app_tick_cnt`

* **Unidad de medida:** Ticks de interrupción del SysTick (donde **1 tick = 1 ms**).
* **Al ejecutar `app_init()`:** Se inicializa exactamente en **`0`** dentro de una sección crítica (con interrupciones globales deshabilitadas).
* **Durante la ejecución de `app_update()`:**
  * En segundo plano, cada 1 ms la interrupción SysTick incrementa el contador (`g_app_tick_cnt++`).
  * En el hilo principal, al comenzar a procesar un ciclo se decrementa (`g_app_tick_cnt--`) para consumir el evento de tiempo.
  * **Comportamiento según la configuración (`TEST_X`):**
    * **En `TEST_0`:** Las tareas tardan solo unos pocos microsegundos en ejecutarse. El ciclo finaliza mucho antes del próximo ms, por lo que `g_app_tick_cnt` oscila habitualmente entre **`0` y `1`**.
    * **En `TEST_1`:** La `task_a_update` bloquea la CPU durante 20 ms (`HAL_Delay(20)`). Durante ese bloqueo ocurren 20 interrupciones de SysTick, haciendo que `g_app_tick_cnt` se incremente hasta **`20`** (o el saldo restante según las iteraciones consumidas). Al salir de la tarea bloqueante, el bucle `while(b_time_update_required)` procesará ciclos de tareas de forma consecutiva para recuperar el retraso acumulado.

---

### `g_app_runtime_us`

* **Unidad de medida:** Microsegundos ($\mu	ext{s}$).
* **Al ejecutar `app_init()`:** No se explicita un valor inicial previo, pero al comenzar la primera iteración de `app_update()` se asigna en **`0`**.
* **Durante la ejecución de `app_update()`:**
  * Al inicio de cada tick procesado en `app_update()`, la variable se reinicia a **`0`**.
  * A medida que las tareas se ejecutan, acumula la suma de los tiempos de ejecución de cada tarea ($	ext{LET}_{	ext{Task A}} + 	ext{LET}_{	ext{Task B}}$).
  * **Comportamiento según la configuración (`TEST_X`):**
    * **En `TEST_0`:** Toma un valor de unos pocos microsegundos (típicamente entre **`1 µs` y `15 µs`**, dependiendo de la frecuencia de reloj del microcontrolador).
    * **En `TEST_1`:** Como la Tarea A genera un retardo bloqueante de 20 ms, la variable tomará un valor cercano a **`20 000 µs`** (más la fracción correspondiente a la Tarea B).

---

### `task_dta_list[index].WCET`

* **Unidad de medida:** Microsegundos ($\mu	ext{s}$) — *Worst-Case Execution Time* (Peor Tiempo de Ejecución Registrado).
* **Al ejecutar `app_init()`:** Se inicializa en **`0`** (`TASK_X_WCET_INI`) para cada elemento del arreglo de tareas.
* **Durante la ejecución de `app_update()`:**
  * **Primera iteración de `app_update()`:** Cada tarea finaliza con un tiempo transcurrido $	ext{LET} > 0$. Al evaluar la condición `if (task_dta_list[index].WCET < task_dta_list[index].LET)`, `WCET` adopta inmediatamente el valor del primer `LET` medido.
  * **Sucesivas iteraciones:** La variable únicamente actualizará su valor incrementándose si en algún ciclo posterior la tarea experimenta un tiempo de ejecución `LET` mayor al máximo registrado hasta ese momento. De lo contrario, conservará su valor histórico.
    * **`task_dta_list[0].WCET` (Tarea A, `index = 0`):** En `TEST_0`, se estabilizará en el tiempo máximo que toma ejecutar el bucle `for` de 1000 iteraciones (aprox. **`10 µs` - `15 µs`**). En `TEST_1`, se estabilizará en aprox. **`20 000 µs`**.
    * **`task_dta_list[1].WCET` (Tarea B, `index = 1`):** Se estabilizará en el tiempo máximo alcanzado por `task_b_update` al ejecutar su lógica (aprox. **`1 µs` - `3 µs`**).