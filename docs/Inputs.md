# Practica 2: Inputs
## 1) Resumen

Nombre del proyecto: Prácticas de Compuertas Lógicas y Secuencias con Raspberry Pi Pico 2

Equipo / Autor(es): Nombre del estudiante / grupo

Curso / Asignatura: Sistemas Embebidos / Microcontroladores

Fecha: DD/MM/AAAA

Descripción breve:
Conjunto de programas que utilizan la Raspberry Pi Pico 2 para manejar entradas digitales (botones) y salidas (LEDs), implementando:

Visualización de las compuertas básicas AND, OR y XOR con dos botones como entradas.

Un selector cíclico de 4 LEDs controlado con botones de avance y retroceso, con lógica de antirrebote por flanco.

## 2) Objetivos
General:

Aprender el manejo de entradas digitales con resistencias pull-up internas y el uso de lógica booleana en MicroPython para controlar múltiples LEDs.

Específicos:

Implementar compuertas lógicas básicas (AND, OR, XOR) con dos botones como entradas.

Mostrar el resultado de las tres operaciones en paralelo con LEDs.

Programar un selector cíclico de LEDs con botones de avance y retroceso.

Incluir lógica de antirrebote por flanco en botones.

## 3) Alcance y Exclusiones
Incluye:

Código en MicroPython para ambos ejercicios.

Esquemático básico de conexiones (botones y LEDs).

Documentación en Markdown para página o repositorio.

No incluye:

Implementación de compuertas con hardware discreto (solo emulación por software).

Variantes en C/C++.

Diseño de PCB o simulación.

## 4) Requisitos
Software:

Thonny IDE o uPyCraft.

Firmware MicroPython en Raspberry Pi Pico 2.

Hardware
Componente|	Cant.	|Nota
Raspberry Pi Pico 2	|1	|MCU principal
LED rojo	|7	|3 para compuertas, 4 para selector
Resistencias 1 kΩ	|7	|Limitadoras de corriente
Botones pulsadores	|4|	A, B, Avanza, Retrocede
Protoboard	|1|	Conexión rápida
Cables Dupont	|Varios|	Macho–macho

Conocimientos previos:

Operaciones lógicas (AND, OR, XOR).

Manejo de interrupciones o lectura por polling de botones.

Antirrebote por software (detección de flanco).

Programación básica en MicroPython.

## 5) Desarrollo
### 5.1 Compuertas básicas AND / OR / XOR con 2 botones

Entradas: Botones A y B, configurados con pull-up (reposo=1, presionado=0).

Salidas: Tres LEDs que muestran en tiempo real el resultado de las operaciones:

LED1 = A AND B

LED2 = A OR B

LED3 = A XOR B

Demostración en video: Se deben probar las 4 combinaciones de entradas: (00, 01, 10, 11).
#### Esquematico
<img src="../recursos/imgs/Esquematicocamino.png" alt="Diagrama del sistema" width="420">
#### Codigo
``` codigo
#include <stdio.h>
#include "pico/stdlib.h"
#include "hardware/timer.h"
#include "hardware/gpio.h"

// Pines de LEDs de la cancha
#define LED0 9
#define LED1 10
#define LED2 11
#define LED3 12
#define LED4 13

// LEDs de puntuación
#define LED_SCORE_P1 7
#define LED_SCORE_P2 8

// Botones
#define BTN_P1 14
#define BTN_P2 15

// Variables del juego
volatile int pelota_pos = 2;       // posición inicial (LED del centro)
volatile int direccion = 1;        // 1 → derecha, -1 → izquierda
volatile bool esperando_respuesta = false;

// Prototipos
bool mover_pelota(struct repeating_timer *t);
void actualizar_leds();
void btn_callback(uint gpio, uint32_t events);

int main() {
    stdio_init_all();

    // Configuración LEDs
    int leds[] = {LED0, LED1, LED2, LED3, LED4, LED_SCORE_P1, LED_SCORE_P2};
    for (int i = 0; i < 7; i++) {
        gpio_init(leds[i]);
        gpio_set_dir(leds[i], true);
        gpio_put(leds[i], 0);
    }

    // Configuración botones
    // Configuración botón jugador 1
gpio_init(BTN_P1);
gpio_set_dir(BTN_P1, false);
gpio_pull_up(BTN_P1);

// Configuración botón jugador 2
gpio_init(BTN_P2);
gpio_set_dir(BTN_P2, false);
gpio_pull_up(BTN_P2);

// Registrar el callback global UNA sola vez
gpio_set_irq_enabled_with_callback(BTN_P1, 0x8u, true, &btn_callback);

// Habilitar interrupción también en el otro botón, sin registrar callback otra vez
gpio_set_irq_enabled(BTN_P2, 0x8u, true);


    // Timer para mover la pelota
    struct repeating_timer timer;
    add_repeating_timer_ms(500, mover_pelota, NULL, &timer);

    while (true) {
        tight_loop_contents(); // loop vacío, todo se maneja con interrupciones y timer
    }
}

// Timer: mueve la pelota
bool mover_pelota(struct repeating_timer *t) {
    if (esperando_respuesta) return true; // espera al jugador

    // Apagar LEDs
    for (int i = LED0; i <= LED4; i++) gpio_put(i, 0);

    // Mover pelota
    pelota_pos += direccion;

    // Revisar si llegó al extremo
    if (pelota_pos <= 0) {
        esperando_respuesta = true; // espera botón del jugador 1
        pelota_pos = 0;
    } else if (pelota_pos >= 4) {
        esperando_respuesta = true; // espera botón del jugador 2
        pelota_pos = 4;
    }

    actualizar_leds();
    return true;
}

// Encender LED de la pelota
void actualizar_leds() {
    gpio_put(LED0, pelota_pos == 0);
    gpio_put(LED1, pelota_pos == 1);
    gpio_put(LED2, pelota_pos == 2);
    gpio_put(LED3, pelota_pos == 3);
    gpio_put(LED4, pelota_pos == 4);
}

// Callback de botones (interrupciones)
void btn_callback(uint gpio, uint32_t events) {
    if (!esperando_respuesta) return;

    if (gpio == BTN_P1 && pelota_pos == 0) {
        direccion = 1; // devuelve hacia la derecha
        esperando_respuesta = false;
    } 
    else if (gpio == BTN_P2 && pelota_pos == 4) {
        direccion = -1; // devuelve hacia la izquierda
        esperando_respuesta = false;
    } 
    else {
        // Falló → punto para el contrario
        if (gpio == BTN_P1) {
            gpio_put(LED_SCORE_P2, 1);
            sleep_ms(500);
            gpio_put(LED_SCORE_P2, 0);
        } else {
            gpio_put(LED_SCORE_P1, 1);
            sleep_ms(500);
            gpio_put(LED_SCORE_P1, 0);
        }
        // Reiniciar pelota
        pelota_pos = 2;
        direccion = (gpio == BTN_P1) ? 1 : -1;
        esperando_respuesta = false;
    }
    actualizar_leds();
}

``` 
#### Video
[Video Compuertas AND / OR / XOR](https://youtube.com/shorts/Svp_Ctx2cuk?si=vYLxuer2qodYCQAW)
### 5.2 Selector cíclico de 4 LEDs con avance/retroceso

Entradas:

Botón AVANZA → incrementa el índice (0 → 1 → 2 → 3 → 0).

Botón RETROCEDE → decrementa el índice (0 → 3 → 2 → 1 → 0).

Salida: Solo un LED encendido a la vez entre LED0..LED3.

Condición: Cada pulsación cuenta solo una vez (antirrebote por flanco). Mantener presionado no genera múltiples avances.

Demostración en video: Se recorre en ambos sentidos mostrando el funcionamiento correcto.
#### Codigo
``` codigo
#include <stdio.h>
#include "pico/stdlib.h"
#include "hardware/timer.h"
#include "hardware/gpio.h"

// Pines de LEDs de la cancha
#define LED0 9
#define LED1 10
#define LED2 11
#define LED3 12
#define LED4 13

// LEDs de puntuación
#define LED_SCORE_P1 7
#define LED_SCORE_P2 8

// Botones
#define BTN_P1 14
#define BTN_P2 15

// Variables del juego
volatile int pelota_pos = 2;       // posición inicial (LED del centro)
volatile int direccion = 1;        // 1 → derecha, -1 → izquierda
volatile bool esperando_respuesta = false;

// Prototipos
bool mover_pelota(struct repeating_timer *t);
void actualizar_leds();
void btn_callback(uint gpio, uint32_t events);

int main() {
    stdio_init_all();

    // Configuración LEDs
    int leds[] = {LED0, LED1, LED2, LED3, LED4, LED_SCORE_P1, LED_SCORE_P2};
    for (int i = 0; i < 7; i++) {
        gpio_init(leds[i]);
        gpio_set_dir(leds[i], true);
        gpio_put(leds[i], 0);
    }

    // Configuración botones
    // Configuración botón jugador 1
gpio_init(BTN_P1);
gpio_set_dir(BTN_P1, false);
gpio_pull_up(BTN_P1);

// Configuración botón jugador 2
gpio_init(BTN_P2);
gpio_set_dir(BTN_P2, false);
gpio_pull_up(BTN_P2);

// Registrar el callback global UNA sola vez
gpio_set_irq_enabled_with_callback(BTN_P1, 0x8u, true, &btn_callback);

// Habilitar interrupción también en el otro botón, sin registrar callback otra vez
gpio_set_irq_enabled(BTN_P2, 0x8u, true);


    // Timer para mover la pelota
    struct repeating_timer timer;
    add_repeating_timer_ms(500, mover_pelota, NULL, &timer);

    while (true) {
        tight_loop_contents(); // loop vacío, todo se maneja con interrupciones y timer
    }
}

// Timer: mueve la pelota
bool mover_pelota(struct repeating_timer *t) {
    if (esperando_respuesta) return true; // espera al jugador

    // Apagar LEDs
    for (int i = LED0; i <= LED4; i++) gpio_put(i, 0);

    // Mover pelota
    pelota_pos += direccion;

    // Revisar si llegó al extremo
    if (pelota_pos <= 0) {
        esperando_respuesta = true; // espera botón del jugador 1
        pelota_pos = 0;
    } else if (pelota_pos >= 4) {
        esperando_respuesta = true; // espera botón del jugador 2
        pelota_pos = 4;
    }

    actualizar_leds();
    return true;
}

// Encender LED de la pelota
void actualizar_leds() {
    gpio_put(LED0, pelota_pos == 0);
    gpio_put(LED1, pelota_pos == 1);
    gpio_put(LED2, pelota_pos == 2);
    gpio_put(LED3, pelota_pos == 3);
    gpio_put(LED4, pelota_pos == 4);
}

// Callback de botones (interrupciones)
void btn_callback(uint gpio, uint32_t events) {
    if (!esperando_respuesta) return;

    if (gpio == BTN_P1 && pelota_pos == 0) {
        direccion = 1; // devuelve hacia la derecha
        esperando_respuesta = false;
    } 
    else if (gpio == BTN_P2 && pelota_pos == 4) {
        direccion = -1; // devuelve hacia la izquierda
        esperando_respuesta = false;
    } 
    else {
        // Falló → punto para el contrario
        if (gpio == BTN_P1) {
            gpio_put(LED_SCORE_P2, 1);
            sleep_ms(500);
            gpio_put(LED_SCORE_P2, 0);
        } else {
            gpio_put(LED_SCORE_P1, 1);
            sleep_ms(500);
            gpio_put(LED_SCORE_P1, 0);
        }
        // Reiniciar pelota
        pelota_pos = 2;
        direccion = (gpio == BTN_P1) ? 1 : -1;
        esperando_respuesta = false;
    }
    actualizar_leds();
}

```
#### Video
[Video Selector Ciclico](https://youtube.com/shorts/twmNGeeP-nU?si=mZBkW5TyqbBxRJ5B)
