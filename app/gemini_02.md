# Análisis del Sistema y del Impacto de `LOGGER_INFO()`

## 1. Análisis del Funcionamiento del Código Fuente (`app.c`, `logger.c`, `logger.h`)

### `logger.h` y `logger.c` (Módulo de Log / Depuración)
* **Funcionalidad principal:** Proveen macros e infraestructura para la impresión de mensajes de depuración/información en consola.
* **Manejo de Semihosting:** Cuando `LOGGER_CONFIG_USE_SEMIHOSTING` está configurado en `1`, la función `logger_log_print_()` ejecuta `printf()` seguido de `fflush(stdout)`. El semihosting permite que la placa (STM32) interactúe con la PC del desarrollador enviando texto por la interfaz del depurador (JTAG/SWD).
* **Mecanismo de Exclusión Mutua / Sección Crítica:** La macro `LOGGER_INFO(...)`:
  1. Ejecuta `__asm("CPSID i")` para **deshabilitar globalmente las interrupciones**.
  2. Formatea la cadena mediante `snprintf(...)` en el búfer `logger_msg`.
  3. Imprime la etiqueta `"[info] "`, el mensaje formateado y un salto de línea `"
"`.
  4. Ejecuta `__asm("CPSIE i")` para **rehabilitar las interrupciones**.

### `app.c` (Planificador y Bucle Principal)
* **`app_init()`:**
  * Realiza llamados a `LOGGER_INFO()` para imprimir datos informativos del inicio del sistema y de las tareas.
  * Inicializa el contador de ciclos DWT (`cycle_counter_init()`).
  * Recorre las tareas (`task_a` y `task_b`), llamando a su función `task_init()` e inicializando su estructura de datos (`NOE=0`, `LET=0`, `BCET=1000`, `WCET=0`).
  * Inicializa las interrupciones de la aplicación (`app_it_init()`).
* **`app_update()`:**
  * Revisa de forma atómica si hay ticks pendientes (`g_app_tick_cnt > 0`).
  * Mientras haya ticks acumulados (`b_time_update_required == true`), ejecuta secuencialmente `task_a_update()` y `task_b_update()`.
  * Mide con DWT el tiempo transcurrido (`LET`) de cada tarea, actualiza sus métricas (`NOE`, `BCET`, `WCET`) y acumula el tiempo total del tick en `g_app_runtime_us`.
  * Al finalizar las tareas del tick, entra en modo *Sleep* (`WFI`).

---

## 2. Impacto de usar `LOGGER_INFO()` en las Variables del Sistema

El impacto de `LOGGER_INFO()` varía drásticamente según **dónde** se utilice y si **`LOGGER_CONFIG_ENABLE`** y **`LOGGER_CONFIG_USE_SEMIHOSTING`** se encuentran activados (set en `1`).

> **Nota sobre Semihosting:** Las operaciones de semihosting imponen un retardo temporal extremadamente alto (del orden de cientos de microsegundos a varias decenas de milisegundos por mensaje) debido a que la CPU se detiene mientras transfiere datos mediante la interfaz JTAG/SWD a la PC. Además, la macro `LOGGER_INFO()` deshabilita las interrupciones durante toda la operación.

---

### A. Impacto en `g_app_tick_cnt`

* **Unidad de medida:** Cantidad de Ticks (donde **1 tick = 1 ms**).
* **Comportamiento normal (sin `LOGGER_INFO` dentro del bucle de tareas):** `g_app_tick_cnt` oscila entre `0` y `1` (en `TEST_0`), ya que el tiempo de ejecución de las tareas es menor a 1 ms.
* **Impacto al usar `LOGGER_INFO()`:**
  1. **Sección Crítica (Deshabilitación de Interrupciones):** `LOGGER_INFO()` ejecuta `CPSID i`. Mientras se formatea y envía el texto, la interrupción `SysTick` queda bloqueada/en espera.
  2. **Retardo por Semihosting:** Si la transmisión por semihosting tarda, por ejemplo, **15 ms**, el procesador estará bloqueado 15 ms con interrupciones deshabilitadas. El periférico SysTick continuará contando en hardware y marcará su bandera de pendiente (*pending status*). Sin embargo, solo podrá registrar **1 interrupción pendiente** a la vez. Las interrupciones del SysTick adicionales que ocurran mientras la interrupción esté deshabilitada se **perderán**.
  3. **Evolución de la variable:**
     * **Si `LOGGER_INFO()` se ejecuta dentro de las tareas (`task_x_update`):** En cada ciclo del bucle principal, `g_app_tick_cnt` tenderá a incrementarse drásticamente si se habilitan interrupciones o, peor aún, se perderán ticks de 1 ms. Al salir de la sección crítica, el SysTick que quedó pendiente incrementará `g_app_tick_cnt++` (acumulando ticks no procesados). El planificador detectará `g_app_tick_cnt > 0` y ejecutará múltiples iteraciones seguidas para tratar de compensar el retraso, alterando el determinismo temporal del sistema.
     * **Si `LOGGER_INFO()` solo se ejecuta en `app_init()`:** No afecta la evolución periódica de `g_app_tick_cnt` durante el bucle principal (`app_update()`), dado que `g_app_tick_cnt` se fuerza a `0` en `app_it_init()` al finalizar la inicialización.

---

### B. Impacto en `g_app_runtime_us`

* **Unidad de medida:** Microsegundos ($\mu	ext{s}$).
* **Comportamiento normal:** Acumula la suma del tiempo de ejecución (`LET`) de la Tarea A y la Tarea B en el tick actual ($pprox 2 	ext{ a } 15\,\mu	ext{s}$ en `TEST_0`).
* **Impacto al usar `LOGGER_INFO()`:**
  * **Si `LOGGER_INFO()` se incluye DENTRO de `task_a_update()` o `task_b_update()`:**
    El tiempo consumido por `LOGGER_INFO()` (formateo `snprintf` + transmisión por semihosting) será medido por el DWT (`cycle_counter_get_time_us()`) como parte del `LET` de esa tarea. Por lo tanto, `g_app_runtime_us` **aumentará drásticamente**, pasando de unos pocos microsegundos a **miles o decenas de miles de microsegundos** ($	ext{ej. } 5\,000 	ext{ a } 50\,000\,\mu	ext{s}$).
  * **Si `LOGGER_INFO()` se incluye en `app_update()` FUERA del bloque medido:** No sumará directamente a `g_app_runtime_us`, pero retrasará el inicio del siguiente ciclo de trabajo.

---

### C. Impacto en `task_dta_list[index].WCET`

* **Unidad de medida:** Microsegundos ($\mu	ext{s}$) — *Worst-Case Execution Time* (Peor Tiempo de Ejecución Registrado).
* **Al ejecutar `app_init()`:** Se inicializa en **`0 µs`** para todas las tareas.
* **Impacto al usar `LOGGER_INFO()`:**
  * **Si `LOGGER_INFO()` se incluye dentro de una tarea (ej. `task_a_update`):**
    En la primera iteración donde se invoque el log, la tarea registrará un `LET` artificialmente inflado por la demora de `snprintf` y la transmisión del semihosting. Dado que `WCET < LET`, la variable `task_dta_list[index].WCET` capturará ese valor inflado (ej. **$pprox 15\,000\,\mu	ext{s}$ a $50\,000\,\mu	ext{s}$**).
  * **Conclusión sobre `WCET`:** La métrica de peor tiempo de ejecución dejará de ser representativa del algoritmo real de la tarea, pasando a medir principalmente la sobrecarga (*overhead*) del sistema de log por semihosting.

---

## 3. Cuadro Resumen del Impacto de `LOGGER_INFO()`

| Variable | Unidad de Medida | Inicialización (`app_init`) | Evolución Normal (`app_update`) | Impacto si se usa `LOGGER_INFO()` en las tareas |
| :--- | :--- | :--- | :--- | :--- |
| **`g_app_tick_cnt`** | Ticks (1 tick = 1 ms) | `0` | Oscila normalmente entre `0` y `1`. | **Se incrementa / acumula retrasos o pierde ticks** debido a la deshabilitación prolongada de interrupciones (`CPSID i`) y el tiempo de semihosting. |
| **`g_app_runtime_us`** | Microsegundos ($\mu	ext{s}$) | Se setea en `0` al inicio de cada tick procesado. | Típicamente entre `2 µs` y `15 µs` (en `TEST_0`). | **Se dispara a miles de $\mu	ext{s}$** ($pprox 5\,000 	ext{ a } 50\,000\,\mu	ext{s}$), pues absorbe el tiempo de ejecución del log. |
| **`task_dta_list[index].WCET`** | Microsegundos ($\mu	ext{s}$) | `0` | Se estabiliza en el tiempo máximo real de la tarea (ej. `10 µs`). | **Queda sobredimensionado**, reflejando el *overhead* de la impresión por semihosting en lugar del cómputo de la tarea. |