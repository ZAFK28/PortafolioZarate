# Comunicación UART bidireccional con control de LED y botón (Raspberry Pi Pico)

> Este proyecto implementa una **comunicación UART bidireccional** entre dos Raspberry Pi Pico, donde una envía datos al presionar un botón, y la otra responde encendiendo o apagando un LED según el comando recibido.  
> **Advertencia:** No conectar directamente a la red eléctrica.

---

## 1) Resumen

- **Nombre del proyecto:** _Control de LED por UART bidireccional_  
- **Autor:** _Rodrigo Zárate_  
- **Asignatura:** _Sistemas Embebidos_  
- **Fecha:** _28/10/2025_  
- **Descripción breve:** Sistema de comunicación serial entre dos Raspberry Pi Pico mediante UART, donde un botón envía comandos y el receptor ejecuta acciones como encender o apagar un LED, además de enviar mensajes de retroalimentación.

!!! tip "Información del proyecto:"
    **Lenguaje/SDK:** C++ con Raspberry Pi Pico SDK (`pico/stdlib.h`, `hardware/uart.h`).  
    **Técnicas clave:** Comunicación UART, lectura digital de botón, control GPIO, procesamiento de cadenas y comandos.  
    **Plataforma:** Raspberry Pi Pico / Pico 2.  

### Material utilizado
- 2 × Raspberry Pi Pico (o Pico 2)  
- 2 × Cables USB  
- 1 × Botón (pushbutton)  
- 1 × LED + resistencia de 330 Ω  
- 1 × Resistencia de 10 kΩ (pull-up)  
- Protoboard y cables jumper macho-macho  

---

## 2) Objetivos

- Configurar una comunicación **UART** confiable entre dos microcontroladores.  
- Enviar comandos de texto como `"LEDON"` y `"LEDOFF"` a través de UART.  
- Detectar la presión de un botón y enviar mensajes automáticos.  
- Controlar un LED remoto en función de los comandos recibidos.  
- Mostrar eco y retroalimentación por consola serial.

---

## 3) Conexiones / Esquema

**Mapeo de pines según el código principal:**

| Señal | Pico (TX) | Pico (RX) | Descripción |
|--------|------------|-----------|--------------|
| `TX` | GP0 | — | Transmisión UART |
| `RX` | — | GP1 | Recepción UART |
| `GND` | GND | GND | Tierra común |

**Entradas y salidas:**

| Dispositivo | GPIO | Función |
|--------------|------|----------|
| Botón | 16 | Entrada digital con `gpio_pull_up()` |
| LED | 15 | Salida digital de control |

**Notas de conexión:**
- Conectar **GP0 (TX)** de una Pico al **GP1 (RX)** de la otra.  
- Compartir **GND** entre ambas placas.  
- Conectar LED en serie con resistencia de 330 Ω al pin 15.  
- Botón conectado a GP16 con resistencia pull-up interna.

**Diagrama de conexión**  
![UART Bidireccional](../recursos/imgs/uart_bidirectional_diagram.png)

---

## 4) Código principal

```cpp
#include "pico/stdlib.h"
#include "hardware/uart.h"
#include <stdio.h>
#include <string>

#define UART_ID uart0
#define BAUD_RATE 115200
#define TX_PIN 0
#define RX_PIN 1
#define button_pin 16
#define led_PIN 15
using namespace std;

int main() {
    stdio_init_all();

    gpio_set_function(TX_PIN, GPIO_FUNC_UART);
    gpio_set_function(RX_PIN, GPIO_FUNC_UART);

    uart_init(UART_ID, BAUD_RATE);
    uart_set_format(UART_ID, 8, 1, UART_PARITY_NONE);

    gpio_init(button_pin);
    gpio_set_dir(button_pin, GPIO_IN);
    gpio_pull_up(button_pin);

    gpio_init(led_PIN);
    gpio_set_dir(led_PIN, GPIO_OUT);

    string c = "";
    string p = "";
    int a = 1;

    while (true) {

        int ch = getchar_timeout_us(0);
        if (ch != PICO_ERROR_TIMEOUT) {
            printf("Eco: %c\n", (char)ch);
            p += (char)ch;
            
            if (ch == '.' || ch == '\n') {
                uart_puts(UART_ID, p.c_str());
                p = "";
            }
        }

        if (gpio_get(button_pin) == 0 && a == 1) {
            printf("Button pressed!\n");
            uart_puts(UART_ID, "LEDON\n");
            sleep_ms(200);
        }
        a = gpio_get(button_pin);

        if (uart_is_readable(uart0)) {
            char character = uart_getc(uart0);
            printf("%c\n", character);
           
            if (character == '\n' || character == '.') {
                if (c == "LEDON") {
                    gpio_put(led_PIN, 1);
                    printf("LED is ON\n");
                }
                else if (c == "LEDOFF") {
                    gpio_put(led_PIN, 0);
                    printf("LED is OFF\n");
                }
                else if (c == "Invalid Command") {
                    printf("Invalid Command\n");
                }
                else {
                    uart_puts(UART_ID, "Invalid Command\n");
                }
                c = "";
                continue;
            }
            else {
                c += character;
            }
        }
    }
}
```
## 5) Explicacion del codigo

### a) Comunicación UART
- Se utiliza **UART0** con los pines GP0 (TX) y GP1 (RX).  
- La velocidad de transmisión es **115200 bps**, ideal para pruebas de eco y envío de cadenas.  
- `uart_puts()` envía cadenas completas; `uart_getc()` recibe un carácter a la vez.

### b) Lógica del botón
- El botón está configurado en **GP16** con `gpio_pull_up()`.  
- Cuando se detecta una presión (lectura baja), se envía el comando `"LEDON\n"` por UART.  
- Se incluye `sleep_ms(200)` para evitar rebotes y envíos múltiples.

### c) Lógica del LED
- Al recibir los comandos `"LEDON"` o `"LEDOFF"`, el microcontrolador enciende o apaga el LED (GPIO 15).  
- Si recibe un mensaje desconocido, envía `"Invalid Command\n"` como retroalimentación.

### d) Mecanismo de eco
- Cualquier carácter recibido desde teclado se reenvía (`printf("Eco: %c\n")`), útil para depuración en consola.

### e) Consideraciones generales
- Ambas Raspberry Pico deben compartir **GND común**.  
- Los comandos deben terminar con **punto (.) o salto de línea (\n)** para ser interpretados.  
- La comunicación es **bidireccional**, permitiendo confirmación del estado.

## 6) Pruebas y comportamiento esperado
- **Caso 1:** Al presionar el botón, se envía `LEDON\n`. El otro dispositivo enciende el LED y responde `"LED is ON"`.  
- **Caso 2:** Al escribir manualmente `LEDOFF` en la terminal, el LED se apaga.  
- **Caso 3:** Si se envía texto distinto, se recibe `"Invalid Command"`.  
- **Caso 4:** Si se desconecta GND, no hay respuesta ni encendido del LED.  
- **Verificación:** En la consola se muestran los mensajes de eco y recepción en tiempo real.  

**Resultado esperado:** Comunicación UART estable y bidireccional con control remoto de un LED mediante botón físico y comandos de texto.

<div style="position:relative;padding-bottom:56.25%;height:0;overflow:hidden;">
  <iframe
    src="https://youtube.com/mNmajwbFeJI?si=hKylJSvT9iahGPBU"
    title="YouTube video"
    allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share"
    allowfullscreen
    style="position:absolute;top:0;left:0;width:100%;height:100%;border:0;">
  </iframe>
</div>