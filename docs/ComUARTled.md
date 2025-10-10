# Comunicación UART para prender un LED (Raspberry Pi Pico)

> Este proyecto utiliza **Comunicación UART** para comunicarse entre dos Raspberry Pi Pico 2.  
> Su objetivo es presionar un botón conectado a una Raspberry y que ésta mande una señal a la otra Raspberry para que prenda un LED.  
> **No conectar a la red eléctrica.**

---

## 1) Resumen

- **Nombre del proyecto:** _Prender un LED por comunicación UART_  
- **Equipo / Autor(es):** _Rodrigo Zárate_  
- **Curso / Asignatura:** _Sistemas Embebidos_  
- **Fecha:** _12/10/2025_  
- **Descripción breve:** _Comunicación serial UART entre dos Raspberry Pi Pico donde una actúa como transmisor (envía datos al presionar un botón) y la otra como receptor (enciende un LED al recibir el dato)._  

!!! tip "Información del proyecto:"
    **Lenguaje/SDK:** C con Raspberry Pi Pico SDK (`pico/stdlib.h`, `hardware/uart.h`, `hardware/gpio.h`).  
    **Técnicas clave:** Comunicación UART asíncrona, transmisión y recepción de bytes, control digital de GPIO.  
    **Plataforma:** Raspberry Pi Pico / Pico 2.  

### Material utilizado
- 2 × Raspberry Pi Pico (o Pico 2)  
- 2 × Cables USB  
- 1 × Botón (pushbutton)  
- 1 × Resistencia de 10 kΩ (pull-down)  
- 1 × LED + resistencia de 330 Ω  
- Protoboard y jumpers macho-macho  

---

## 2) Objetivos

- Configurar comunicación **UART** entre dos Raspberry Pi Pico.  
- Programar un **transmisor** que envíe un carácter al presionar un botón.  
- Programar un **receptor** que encienda un **LED** al recibir el carácter.  
- Implementar **baud rate común (9600 bps)** para sincronizar ambas placas.  
- Verificar la correcta transmisión y recepción mediante el encendido del LED.

---

## 3) Conexiones / Esquema

**Mapeo de pines (según el código):**

| Señal | Pico (TX) | Pico (RX) | Descripción |
|--------|------------|-----------|--------------|
| `TX` | GP0 | — | Transmisor UART |
| `RX` | — | GP1 | Receptor UART |
| `GND` | GND | GND | Tierra común entre ambas placas |

**Entradas y salidas:**

| Dispositivo | GPIO | Función |
|--------------|------|----------|
| **Maestro (TX)** | 14 | Botón de envío |
| **Esclavo (RX)** | 15 | LED indicador |

**Notas de conexión:**
- Conectar **GP0 (TX)** del maestro a **GP1 (RX)** del esclavo.  
- Conectar **GND a GND**.  
- Usar resistencia **330 Ω** en serie con el LED.  
- Usar **resistencia de 10 kΩ pull-down** en el botón.

**Diagrama de conexión**  
![UART LED](../recursos/imgs/uart_led_diagram.png)

---

## 4) Código
### Codigo emisor
```codigo
    #include "pico/stdlib.h"
#include "hardware/uart.h"
#include <stdio.h>

#define UART_ID uart0
#define BAUD_RATE 115200
#define TX_PIN 0
#define RX_PIN 1
#define button_pin 15

int main() {
    stdio_init_all();

    gpio_set_function(TX_PIN, GPIO_FUNC_UART);
    gpio_set_function(RX_PIN, GPIO_FUNC_UART);

    uart_init(UART_ID, BAUD_RATE);
    uart_set_format(UART_ID, 8, 1, UART_PARITY_NONE);

    gpio_init(button_pin);
    gpio_set_dir(button_pin, GPIO_IN);
    gpio_pull_up(button_pin);
    while (true){
        if (gpio_get(button_pin) == 0) {
            uart_puts(UART_ID, "1");
            sleep_ms(10);
        }
        else {
            uart_puts(UART_ID, "0");
            sleep_ms(10);
        }
    }
}
```
### Codigo receptor
```codigo
#include "pico/stdlib.h"
#include "hardware/uart.h"
#include <stdio.h>

#define BAUD_RATE 115200
#define TX_PIN 0
#define RX_PIN 1
#define led_PIN 15

int main() {
    stdio_init_all();

    gpio_set_function(TX_PIN, GPIO_FUNC_UART);
    gpio_set_function(RX_PIN, GPIO_FUNC_UART);

    uart_init(uart0, BAUD_RATE);
    uart_set_format(uart0, 8, 1, UART_PARITY_NONE);
    gpio_init(led_PIN);
    gpio_set_dir(led_PIN, GPIO_OUT);

    printf("[SUPPORT] Esperando mensajes (polling)...\n");

    while (true) {
        if (uart_is_readable(uart0)) {
            char c = uart_getc(uart0);
            if (c == '1') {
                gpio_put(led_PIN, 1);
                
            } else {
                gpio_put(led_PIN, 0);
                
            } 
        }
    }
}
```
## 5) Explicación del programa

### a) Comunicacion UART

- Se utiliza UART0 con Tx = GP0 y Rx = GP1.
- La comunicación se realiza a 9600 bps, un valor estable para transmisión entre microcontroladores.
- El transmisor envía un carácter ASCII ('A'), que el receptor interpreta para encender un LED.

### b) Lógica del transmisor

- El botón en GP14 envía 'A' por UART cuando se presiona.
- Usa sleep_ms(300) para evitar rebotes.
- Se transmite un solo byte por evento.

### c) Lógica del receptor

- El receptor monitorea constantemente con uart_is_readable().
- Al recibir 'A', enciende el LED durante 500 ms.
- Esto confirma que la comunicación UART funciona correctamente.

### d) Consideraciones

- Ambas Raspberry deben compartir GND común.
- Si hay errores en transmisión, verificar la velocidad (baud rate).
- Se puede ampliar el sistema para enviar distintos caracteres y controlar más salidas.

---

## 6) Pruebas y comportamiento esperado

- Transmisor: Al presionar el botón, se envía el carácter 'A'.
- Receptor: Enciende el LED al recibir 'A'.
- Verificación: Si se desconecta GND, el LED no responde.
- Prueba adicional: Conectar el receptor a un monitor serial mostrará el carácter recibido.
- Resultado esperado: Comunicación confiable UART entre dos Raspberry Pi Pico, confirmada por el encendido del LED.

<div style="position:relative;padding-bottom:56.25%;height:0;overflow:hidden;">
  <iframe
    src="https://www.youtube.com/watch?v=mNmajwbFeJI"
    title="YouTube video"
    allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share"
    allowfullscreen
    style="position:absolute;top:0;left:0;width:100%;height:100%;border:0;">
  </iframe>
</div>


