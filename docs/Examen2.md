# Control de Servo con Lista de Ángulos (UART + Botones)

> Sistema que permite **cargar** una lista de ángulos para un **servo** (0–180°) mediante **consola/serial**, **reproducirlos** automáticamente o **navegarlos** con botones.  
> **No conectar a la red eléctrica.** El servo debe alimentarse a **5–6 V** y **compartir GND** con la Pico/Pico 2.

---

## 1) Resumen

- **Nombre del proyecto:** _Lista de ángulos para servo por UART y botones_  
- **Equipo / Autor(es):** _Rodrigo Zárate_  
- **Curso / Asignatura:** _Sistemas Embebidos_  
- **Fecha:** _22/10/2025_  
- **Descripción breve:** _La Pico 2 recibe comandos por consola/serial para “Escribir” una lista de ángulos del servo. Con un botón se cambian los **modos** (Escritura / Lectura / Navegación) y con otros dos botones se avanza/retrocede entre ángulos en Navegación. El PWM se configura a ~50 Hz._

!!! tip "Información del proyecto:"
    **Lenguaje/SDK:** C con Raspberry Pi Pico SDK (`pico/stdlib.h`, `hardware/uart.h`, `hardware/pwm.h`, `hardware/gpio.h`).  
    **Técnicas clave:** Parsing sobre **stdio** (consola), **PWM** a 50 Hz para servo, **GPIO con interrupción** para cambio de modo.  
    **Plataforma:** Raspberry Pi Pico / Pico 2.  
    **Baud rate:** **115200 bps**.  

### Material utilizado
- 1 × Raspberry Pi Pico (o Pico 2)  
- 1 × Servo estándar (5–6 V)  
- 3 × Botones (pushbutton)  
- 3 × Resistencias (si no usas pull-ups internas)  
- Jumpers, protoboard  
- Fuente 5–6 V para servo (compartir **GND** con la Pico)  
- Cable USB (consola/serial por USB)

---

## 2) Objetivos

- Configurar **PWM ~50 Hz** para controlar un servo por **ancho de pulso** (1.0–2.0 ms ↔ 0–180°).  
- Implementar **tres modos**:  
  - **Modo 0 – Escritura:** cargar lista de ángulos desde consola.  
  - **Modo 1 – Lectura:** reproducir toda la lista automáticamente.  
  - **Modo 2 – Navegación:** avanzar/retroceder por la lista con botones.  
- Diseñar un **protocolo sencillo** de texto: `Escribir,<a1>,<a2>,...,<an>;` y `Borrar;`.

---

## 3) Conexiones / Esquema

**Mapeo de pines (según el código):**

| Señal / Dispositivo | GPIO | Función |
|---------------------|------|----------|
| **UART0 TX** | GP0 | TX hardware |
| **UART0 RX** | GP1 | RX hardware |
| **Botón Modo** | GP16 | Cambia `programa` (0→1→2→0) |
| **Botón Siguiente** | GP17 | Avanza en Modo Navegación |
| **Botón Anterior** | GP18 | Retrocede en Modo Navegación |
| **Servo (PWM)** | GP15 | Señal PWM (50 Hz) |

**Notas de conexión:**
- Los botones usan **pull-ups internas** (`gpio_pull_up`).  
  Conéctalos de **GPIO → Botón → GND** (activo-bajo).  
- **Servo:** Señal a **GP15**, **+5–6 V** a Vcc del servo, **GND común** con la Pico.  
- Si usas UART físico, respeta **115200 bps** y GND común.  

**Diagrama general (texto):**  
`[Botón Modo GP16] [Botón Next GP17] [Botón Prev GP18] → Pico`  
`Pico GP15 PWM → Servo (señal)` — `Servo Vcc 5–6 V` — `GND común`

---

## 4) Código

```c
#include "pico/stdlib.h"
#include "hardware/uart.h"
#include <stdio.h>
#include <string>
#include "hardware/pwm.h"
#define UART_ID uart0
#define BAUD_RATE 115200
#define TX_PIN 0
#define RX_PIN 1
#define btn_mode 16
#define btn_next 17
#define btn_prev 18
#define PWM_PIN 0
#define SERVO_PIN 15
using namespace std;
#define list 10

volatile int programa = 0;

uint16_t angle_to_pwm(float angle) {
    float pulse_ms = 1.0f + (angle / 180.0f);   // 1.0 → 2.0 ms
    float duty = (pulse_ms / 20.0f) * 39062.0f; // 20 ms periodo total
    return (uint16_t)duty;
}

// --- Rutina de interrupción ---
static void button_isr(uint gpio, uint32_t events) {
    if (gpio == btn_mode && (events & GPIO_IRQ_EDGE_FALL)) {
        programa++;
        if (programa > 2) {
            programa = 0;
        }
        if (programa == 0) {
            printf("Modo de Escritura Activado\n");
        } else if (programa == 1) {
            printf("Modo de Lectura Activado\n");
        } else if (programa == 2) {
            printf("Modo de Navegacion Activado\n");
        }
        sleep_ms(500);
    }
}

int main() {
    int pos = 0;
    stdio_init_all();
    uart_init(UART_ID, BAUD_RATE);
    uart_set_format(UART_ID, 8, 1, UART_PARITY_NONE);
    gpio_set_function(SERVO_PIN, GPIO_FUNC_PWM);
    uint slice = pwm_gpio_to_slice_num(SERVO_PIN);
    pwm_set_clkdiv(slice, 64.0f);
    pwm_set_wrap(slice, 39062);
    pwm_set_enabled(slice, true);

    int next1 = 0;
    int prev1 = 0;

    // Botones con pull-up interna
    gpio_init(btn_mode);
    gpio_set_dir(btn_mode, GPIO_IN);
    gpio_pull_up(btn_mode);

    gpio_init(btn_next);
    gpio_set_dir(btn_next, GPIO_IN);
    gpio_pull_up(btn_next);

    gpio_init(btn_prev);
    gpio_set_dir(btn_prev, GPIO_IN);
    gpio_pull_up(btn_prev);

    // Interrupción botón modo
    gpio_set_irq_enabled_with_callback(btn_mode, GPIO_IRQ_EDGE_FALL, true, &button_isr);

    string p1 = "";
    int lista[list] = {720, 720, 720, 720, 720, 720, 720, 720, 720, 720};
    int i = -1;

    while (true) {
        int pos = 0;

        if (programa == 0) {
            int ch = getchar_timeout_us(0);
            if (ch != PICO_ERROR_TIMEOUT) {

                if (ch == ',') {
                    if (p1 == "Escribir" || p1 == "write" || p1 == "WRITE" || p1 == "escribir" || p1 == "Write" || p1 == "ESCRIBIR") {
                        p1 = "";
                        int contador = 0;
                        for (int k = 0; k < list; k++) {
                            if (lista[k] != 720) {
                                contador++;
                            }
                        }
                        if (contador == list) {
                            printf("Lista llena, no se pueden agregar mas elementos.\n");
                            p1 = "";
                            continue;
                        } else {
                            i++;
                            while (ch != ';' && ch != '\n') {
                                ch = getchar_timeout_us(0);
                                if (ch != PICO_ERROR_TIMEOUT) {
                                    if (ch == ',') {
                                        printf("Eco: %s\n", p1.c_str());
                                        if (stoi(p1) > 180 || stoi(p1) < 0) {
                                            printf("Valor fuera de rango (0-180). Intente de nuevo.\n");
                                            i--;
                                        } else {
                                            lista[i] = stoi(p1);
                                            i++;
                                            p1 = "";
                                        }
                                    } else if (ch == ';' || ch == '\n') {
                                        if (stoi(p1) > 180 || stoi(p1) < 0) {
                                            printf("Valor fuera de rango (0-180). Intente de nuevo.\n");
                                            i--;
                                        } else {
                                            lista[i] = stoi(p1);
                                            p1 = "";
                                        }
                                    } else {
                                        p1 += (char)ch;
                                    }

                                    printf("Lista: ");
                                    for (int k = 0; k <= i; ++k) {
                                        printf("%d", lista[k]);
                                        if (k + 1 <= i) printf(", ");
                                    }
                                    printf("\n");
                                }
                            }
                        }
                    }
                }

                if (ch == ';' || ch == '\n') {
                    if (p1 == "Borrar" || p1 == "delete" || p1 == "DELETE" || p1 == "borrar" || p1 == "BORRAR") {
                        for (int k = 0; k < list; k++) lista[k] = 720;
                        i = -1;
                        printf("Ok, Lista borrada\n");
                    }
                    p1 = "";
                } else {
                    p1 += (char)ch;
                }
            }
        } else if (programa == 1) {
            while (programa == 1) {
                int contador = 0;
                for (int k = 0; k < list; k++) {
                    if (lista[k] != 720) contador++;
                }
                if (contador == 0) {
                    printf("La lista esta vacia. Cambie al modo de escritura.\n");
                    sleep_ms(1500);
                } else {
                    int j = 0;
                    while (programa == 1 && j < contador) {
                        printf("posicion: %d , valor: %d\n", j, lista[j]);
                        pwm_set_gpio_level(SERVO_PIN, angle_to_pwm(lista[j]));
                        printf("Angulo: %.1f°\n", lista[j]);
                        sleep_ms(1500);
                        j++;
                    }
                }
            }
        } else if (programa == 2) {
            while (programa == 2) {
                static int i = 0;
                const int LISTA_LEN = 10;

                if (lista[pos] == 720) pos--;
                int contador = 0;
                for (int k = 0; k < list; k++) if (lista[k] != 720) contador++;

                if (next1 == 0 && gpio_get(btn_next) == 1) {
                    if (contador == 0 && gpio_get(btn_next) == 1) {
                        printf("Lista Vacia, no se puede avanzar.\n");
                        sleep_ms(100);
                    } else {
                        pos++;
                        printf("siguiente \n");
                        if (lista[pos] == 720) {
                            printf("Valor no valido, regresando a %d\n", pos - 1);
                            pos--;
                        }
                        if (pos == list) pos = 0;
                        printf("posicion: %d , valor: %d\n", pos, lista[pos]);
                        pwm_set_gpio_level(SERVO_PIN, angle_to_pwm(lista[pos]));
                        printf("Angulo: %.1f°\n", lista[pos]);
                        sleep_ms(200);
                    }
                }
                next1 = gpio_get(btn_next);

                if (prev1 == 0 && gpio_get(btn_prev) == 1) {
                    if (contador == 0 && gpio_get(btn_prev) == 1) {
                        printf("Lista Vacia, no se puede avanzar.\n");
                        sleep_ms(100);
                    } else {
                        pos--;
                        printf("anterior \n");
                        if (pos == -1) {
                            printf("Valor no valido, regresando a %d\n", pos + 1);
                            pos++;
                        }
                        if (pos == list) pos = 0;
                        printf("posicion: %d , valor: %d\n", pos, lista[pos]);
                        pwm_set_gpio_level(SERVO_PIN, angle_to_pwm(lista[pos]));
                        printf("Angulo: %.1f°\n", lista[pos]);
                        sleep_ms(200);
                    }
                }
                prev1 = gpio_get(btn_prev);
            }
        }
    }
    return 0;
}
```
## 5) Explicación del programa

### a) PWM (Control del Servo)
- La frecuencia del PWM se configura aproximadamente a **50 Hz**, ideal para servos convencionales.  
- Los parámetros clave son:
  - `clkdiv = 64`  
  - `wrap = 39062`  
  Estos valores generan un periodo de **20 ms**.  
- La función `angle_to_pwm(angle)` convierte un ángulo de **0 a 180°** en una señal de **1.0 a 2.0 ms**, correspondiente al rango completo del servo.  
- El cálculo interno es:  
  \[
  \text{ticks} = \frac{\text{pulso\_ms}}{20\text{ ms}} \times 39062
  \]
  obteniendo valores entre **1953** (0°) y **3906** (180°).

---

### b) Modos de operación
El sistema utiliza tres modos definidos por la variable global `programa`, la cual cambia mediante interrupción con el botón `btn_mode` (GPIO 16):

| Modo | Valor | Descripción |
|------|--------|-------------|
| **Modo 0 – Escritura** | 0 | Permite **cargar** una lista de ángulos desde la consola UART. |
| **Modo 1 – Lectura** | 1 | **Reproduce** automáticamente la lista de ángulos almacenados. |
| **Modo 2 – Navegación** | 2 | Permite **avanzar** o **retroceder** por la lista usando los botones físicos. |

Cada modo imprime en consola su nombre al activarse:

- Modo de Escritura Activado

- Modo de Lectura Activado

- Modo de Navegacion Activado


---

### c) Lógica de los modos

#### 🔹 **Modo 0 – Escritura**
- Usa `getchar_timeout_us(0)` para leer texto de la consola.  
- Admite los siguientes comandos:

| Comando | Función |
|----------|----------|
| `Escribir,90,120,45,0,180;` | Carga una lista con los ángulos proporcionados. |
| `Borrar;` | Vacía completamente la lista. |

- Los valores deben estar entre **0 y 180 grados**.  
- Se almacenan en un arreglo `lista[10]` con un máximo de **10 posiciones**.  
- El valor **720** indica posición vacía.  
- Cada vez que se agrega un valor, el sistema imprime la lista actualizada


---

### d) Interrupción de cambio de modo
- El **botón de modo** (GPIO 16) está configurado con **pull-up interno**.  
- Se detecta un **flanco de bajada** (cuando el botón se presiona a GND).  
- La interrupción ejecuta `button_isr()`, que incrementa `programa` (0 → 1 → 2 → 0).  
- Se incluye un `sleep_ms(500)` dentro de la ISR para evitar rebotes, aunque se recomienda migrarlo al bucle principal.

---

### e) Comunicación UART / Consola
- Configurada a **115200 bps**, 8 bits de datos, sin paridad, 1 bit de parada (8N1).  
- El código inicializa UART0 (GPIO 0/1), pero el `stdio` está disponible por **USB CDC**, por lo que se puede usar directamente el **Monitor Serial** del entorno de desarrollo sin cablear TX/RX.  
- Si se desea usar UART físico, conectar **TX (GP0)** y **RX (GP1)** con **GND común**.

---

## 6) Video de demostracion
<div style="position:relative;padding-bottom:56.25%;height:0;overflow:hidden;">
  <iframe
    src="https://www.youtube.com/embed/-4OjMs1xZ-A?rel=0&modestbranding=1&playsinline=1"
    title="YouTube Short"
    allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share"
    allowfullscreen
    style="position:absolute;top:0;left:0;width:100%;height:100%;border:0;">
  </iframe>
</div>


