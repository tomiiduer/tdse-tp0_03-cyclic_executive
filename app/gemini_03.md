# Análisis del Código Fuente: `board.h`, `dwt.h` y `systick.c`

Este informe analiza el funcionamiento y el rol que desempeñan los archivos de soporte del sistema embebido ARM Cortex-M (`board.h`, `dwt.h`, `systick.h` y `systick.c`).

---

## 1. `board.h` (Abstracción de Hardware y Selección de Placa)

El archivo **`board.h`** actúa como una capa de abstracción de hardware (HAL/BSP) que permite compilar el mismo código de aplicación en múltiples placas de desarrollo basadas en microcontroladores STM32.

### Aspectos clave de su funcionamiento:
* **Selección de Placa mediante Macros:** Define identidades numéricas para 10 placas distintas (familias STM32F1, F3, F4 y F7). La macro `BOARD` especifica la placa activa (configurada por defecto en `NUCLEO_F103RC`).
* **Mapeo de Periféricos (LEDs y Botones):** Según la placa seleccionada, mapea alias genéricos como `BTN_A_PIN`, `LED_A_PIN`, `BTN_PRESSED` o `LED_ON` hacia los pines y estados lógicos concretos de la placa activa.
* **Portabilidad:** Permite que las tareas de la aplicación (`task_a`, `task_b`, etc.) manipulen hardware básico (botón del usuario o LED de estado) usando nombres simbólicos uniformes, sin importar si el botón es activo por nivel bajo (`GPIO_PIN_RESET`) o alto (`GPIO_PIN_SET`).

---

## 2. `dwt.h` (Medición del Tiempo de Ejecución mediante Periférico DWT)

El archivo **`dwt.h`** hace uso del módulo **DWT (*Data Watchpoint and Trace*)**, un periférico integrado dentro del núcleo ARM Cortex-M (como ARM Cortex-M3/M4/M7). Implementa funciones `static inline` optimizadas a nivel de ensamblador para medir el tiempo de ejecución en ciclos de reloj y microsegundos sin sobrecarga (*overhead*) considerable.

### Funciones e Interfaz:
* **`cycle_counter_init()`:** Habilita el módulo de rastreo en el registro `DEMCR` (`CoreDebug_DEMCR_TRCENA_Msk`), reinicia a 0 el contador de ciclos `CYCCNT` y habilita el conteo mediante el bit `CYCCNTENA` del registro de control `DWT->CTRL`.
* **`cycle_counter_reset()`:** Pone a cero el contador de ciclos `DWT->CYCCNT`.
* **`cycle_counter_enable()` / `cycle_counter_disable()`:** Habilitan o pausan la cuenta del registrador `CYCCNT`.
* **`cycle_counter_get()`:** Retorna la cantidad de ciclos de reloj de CPU transcurridos (`uint32_t`).
* **`cycle_counter_get_time_us()`:** Convierte el valor actual de `CYCCNT` a microsegundos dividiendo los ciclos acumulados entre la frecuencia del sistema en MHz:
  $$	ext{Tiempo } (\mu	ext{s}) = rac{	ext{DWT->CYCCNT}}{	ext{SystemCoreClock} / 1\,000\,000}$$

### Rol en el sistema:
Permite calcular métricas en tiempo real como `LET` (*Last Execution Time*), `BCET` y `WCET` (*Worst-Case Execution Time*) dentro del planificador `app.c`.

---

## 3. `systick.h` y `systick.c` (Retardo Bloqueante en Microsegundos)

El módulo **`systick`** provee una función para generar retardos (*delays*) precisos en microsegundos aprovechando el temporizador del sistema del núcleo ARM Cortex (**SysTick**).

### Funcionamiento de `systick_delay_us(uint32_t delay_us)`:
* **Obtención del contador actual:** Captura el valor actual del registro de decremento del SysTick (`SysTick->VAL`).
* **Cálculo del objetivo (*target*):** Multiplica la cantidad de microsegundos deseados por los ciclos por microsegundo de la CPU (`delay_us * (SystemCoreClock / 1000000UL)`).
* **Manejo del Desbordamiento (*Rollover*):** El timer SysTick es un contador decreciente que al llegar a 0 se recarga automáticamente desde `SysTick->LOAD`. La función calcula los ciclos transcurridos (`elapsed`) considerando dos escenarios:
  1. **Sin desbordamiento (`current <= start`):** $	ext{elapsed} = 	ext{start} - 	ext{current}$.
  2. **Con desbordamiento (`current > start`):** $	ext{elapsed} = 	ext{SysTick->LOAD} + 	ext{start} - 	ext{current}$.
* **Espera activa (*Polling*):** Mantiene a la CPU en un bucle `while(1)` hasta que $	ext{elapsed} \ge 	ext{target}$.

### Rol en el sistema:
Ofrece una alternativa a `HAL_Delay()` para demoras muy cortas ($\mu	ext{s}$ en lugar de $	ext{ms}$), útil cuando una tarea o controlador requiere un retardo preciso sin depender del handler periódico del SysTick.