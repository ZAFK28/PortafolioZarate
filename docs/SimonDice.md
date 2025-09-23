# Simon Dice
---
## 1) Resumen

Nombre del proyecto: Simón de LEDs con display 7 segmentos
Equipo / Autor(es): Rodrigo Zárate
Curso / Asignatura: Sistemas Embebidos
Fecha: 22/09/2025

Descripción breve:
Juego de memoria (Simon Says) que genera una secuencia aleatoria de LEDs (4 posiciones). El jugador debe repetir la secuencia usando 4 botones con pull-up interno y lectura con debounce. Un display de 7 segmentos (cátodo común) muestra el número de ronda, hay animaciones de inicio/fin y una animación de victoria al completar todas las rondas.

Información del proyecto:

Lenguaje/SDK: C con Raspberry Pi Pico SDK (pico/stdlib.h).

Técnicas clave: manejo de GPIO, debounce por software, generación pseudoaleatoria con siembra por timestamp, lógica de estados por rondas, control de display 7-segmentos por máscara de bits.

Plataforma: Raspberry Pi Pico / Pico 2.

Material utilizado:

Raspberry Pi Pico (o Pico 2) + cable micro-USB/USB-C

Protoboard

4 LEDs (juego) + 4 resistencias 220–330 Ω (una por LED)

4 botones momentáneos + (opcional) 1–2 kΩ si no se usan pull-ups internos

Display 7 segmentos cátodo común (1 dígito) + 8 resistencias 220–330 Ω (una por segmento A–G y DP)

Jumpers

PC con VS Code + Pico SDK configurado

## 2) Objetivos

Leer 4 botones con pull-up interno (activos en bajo) y debounce por software.

Generar y reproducir secuencias pseudoaleatorias de LEDs, incrementando la longitud por ronda.

Mostrar el número de ronda (0–9, A–F) en un display de 7 segmentos mediante máscaras de bits.

Practicar estructura de firmware: inicialización → lazo principal → funciones de utilidad (parpadeo, display, entrada de usuario) → animaciones de estado (win/game-over).

## 3) Conexiones / Esquema
GPIO asignados

LEDs (juego, activos en alto):

LED0 → GPIO 2

LED1 → GPIO 3

LED2 → GPIO 4

LED3 → GPIO 5

Botones (activos en bajo, con gpio_pull_up()):

BTN0 → GPIO 6

BTN1 → GPIO 7

BTN2 → GPIO 8

BTN3 → GPIO 9

Display 7 segmentos (cátodo común):

Orden físico de pines usado en el código: DISPLAY_PIN = {E, D, C, Dp, G, F, A, B}

E → GPIO 12

D → GPIO 13

C → GPIO 14

Dp → GPIO 15

G → GPIO 16

F → GPIO 17

A → GPIO 18

B → GPIO 19

Recomendaciones de conexión:

LEDs: ánodo a GPIO a través de resistencia serie (220–330 Ω); cátodo a GND. gpio_put(pin, 1) enciende.

Display cátodo común: cada segmento (A–G, DP) debe llevar su propia resistencia a la línea del GPIO; cátodo común del display a GND. Con esta configuración, gpio_put(pin, 1) enciende el segmento.

Botones: un terminal a GND y el otro al GPIO correspondiente; habilitar gpio_pull_up().

Tabla rápida de pines

|  Señal	|   GPIO   |          Uso          |
|----------:|:--------:|-----------------------|
|LED0       |	2	   |Botón jugador izquierdo|
|LED1    	|3         |	Posición 3 (centro)|
|LED2	    |4         |	Posición 4         |
|LED3   	|5         |Posición 5(extremo der)|
|BTN0      	|6         | Indicador de punto izq|
|BTN1       |7         | Indicador de punto der|
|BTN2      	|8         | Indicador de punto izq|
|BTN3       |9         | Indicador de punto der|
|E      	|12        | Indicador de punto izq|
|D          |13        | Indicador de punto der|
|C      	|14        | Indicador de punto izq|
|Dp         |15        | Indicador de punto der|
|G      	|16        | Indicador de punto izq|
|F          |17        | Indicador de punto der|
|A      	|18        | Indicador de punto izq|
|B          |19        | Indicador de punto der|

### Esquematico
<img src="../recursos/imgs/pong.png" alt="Diagrama del sistema" width="420">
## 4) Código
``` codigo
#include <stdio.h>
#include "pico/stdlib.h"
#include "hardware/gpio.h"
#include <pico/time.h>
#include <stdlib.h>
 
#define DIS_COUNT     8
#define LED_COUNT     4
#define BTN_COUNT     4
#define MAX_STEPS     15
 
// Pines:
const uint LED_PINS[LED_COUNT] = {2, 3, 4, 5};      // LEDs en GPIO 2–5
const uint BTN_PINS[BTN_COUNT] = {6, 7, 8, 9};      // Botones en GPIO 6–9
// Display 7 seg cátodo común: 12=E, 13=D, 14=C, 15=Dp, 16=G, 17=F, 18=A, 19=B
const uint DISPLAY_PIN[DIS_COUNT] = {12, 13, 14, 15, 16, 17, 18, 19}; // [E,D,C,Dp,G,F,A,B]
 
// Timings (ms)
#define LED_ON_MS       400
#define LED_OFF_MS      150
#define INPUT_LED_MS    200
#define PAUSE_ROUND_MS  500
#define DEBOUNCE_MS     25
#define HOLD_MS         15
 
static uint8_t pattern[MAX_STEPS];
static uint8_t round_len = 0;
 
static void led_on(uint i)  { gpio_put(LED_PINS[i], 1); }
static void led_off(uint i) { gpio_put(LED_PINS[i], 0); }
 
static void blink(uint i, uint on_ms) {
    led_on(i);
    sleep_ms(on_ms);
    led_off(i);
    sleep_ms(LED_OFF_MS);
}
 
static void all_off(void) {
    for (uint i = 0; i < LED_COUNT; i++) led_off(i);
}
 
static void all_blink(uint times, uint on_ms) {
    for (uint t = 0; t < times; t++) {
        for (uint i = 0; i < LED_COUNT; i++) gpio_put(LED_PINS[i], 1);
        sleep_ms(on_ms);
        all_off();
        sleep_ms(LED_OFF_MS);
    }
}
 
// ===== 7-SEG: utilidades =====
// Máscara de segmentos en orden: [A,B,C,D,E,F,G,DP] (bit 0 = A, bit 7 = DP)
static const uint8_t DIGIT_MASKS[16] = {
    /*0*/ 0b00111111,
    /*1*/ 0b00000110,
    /*2*/ 0b01011011,
    /*3*/ 0b01001111,
    /*4*/ 0b01100110,
    /*5*/ 0b01101101,
    /*6*/ 0b01111101,
    /*7*/ 0b00000111,
    /*8*/ 0b01111111,
    /*9*/ 0b01101111,
    /*10*/ 0b01110111, // "A"
    /*11*/ 0b01111100, // "b"
    /*12*/ 0b00111001, // "C"
    /*13*/ 0b01011110, // "d"  
    /*14*/ 0b01111001, // "E"
    /*15*/ 0b01110001  // "F"
};
// Máscara para "U" (win)
#define SEG_U 0b00111110
// Escribe la máscara ABCDEFGDp en pines [E,D,C,Dp,G,F,A,B]
static void seg_write_mask(uint8_t m) {
    // Cátodo común: 1 = ON
    gpio_put(DISPLAY_PIN[0], (m >> 4) & 1); // E
    gpio_put(DISPLAY_PIN[1], (m >> 3) & 1); // D
    gpio_put(DISPLAY_PIN[2], (m >> 2) & 1); // C
    gpio_put(DISPLAY_PIN[3], (m >> 7) & 1); // Dp
    gpio_put(DISPLAY_PIN[4], (m >> 6) & 1); // G
    gpio_put(DISPLAY_PIN[5], (m >> 5) & 1); // F
    gpio_put(DISPLAY_PIN[6], (m >> 0) & 1); // A
    gpio_put(DISPLAY_PIN[7], (m >> 1) & 1); // B
}
 
static void seg_clear(void) {
    for (int i = 0; i < DIS_COUNT; i++) gpio_put(DISPLAY_PIN[i], 0);
}
 
static void seg_show_digit(int d, bool dp) {
    if (d < 0 || d > 15) { seg_clear(); return; }
    uint8_t m = DIGIT_MASKS[d];
    seg_write_mask(m);
}
 
static void seg_show_mask(uint8_t m) { seg_write_mask(m); }
 
// ===== Botones =====
static int read_button_once(void) {
    for (int i = 0; i < BTN_COUNT; i++) {
        if (gpio_get(BTN_PINS[i]) == 0) {      // activo en bajo (pull-up)
            sleep_ms(DEBOUNCE_MS);
            if (gpio_get(BTN_PINS[i]) == 0) {
                sleep_ms(HOLD_MS);
                while (gpio_get(BTN_PINS[i]) == 0) tight_loop_contents();
                return i;
            }
        }
    }
    return -1;
}
 
static int wait_button_blocking(void) {
    int b = -1;
    while (b < 0) {
        b = read_button_once();
        tight_loop_contents();
    }
    return b;
}
 
// ===== Aleatoriedad =====
static inline uint8_t randi4(void) { return (uint8_t)(rand() & 3); }
 
// Espera a la PRIMERA pulsación para iniciar y siembra RNG con su timestamp
static void wait_for_start_and_seed(void) {
    // Modo espera: display en 0, “tick” de LEDs hasta pulsar
    seg_show_digit(0, false);
    uint32_t idx = 0;
    while (1) {
        // mini animación para que se note que está “vivo”
        blink(idx, 80);
        idx = (idx + 1) % LED_COUNT;
 
        int b = read_button_once();
        if (b >= 0) {
            // Semilla con el instante de la pulsación
            uint64_t t = time_us_64();
            unsigned seed = (unsigned)(t ^ (t >> 32) ^ (b * 2654435761u));
            srand(seed);
            // breve pausa para no tomar esta pulsación como input de juego
            sleep_ms(200);
            return;
        }
    }
}
 
// ===== Juego =====
static void play_sequence(void) {
    sleep_ms(PAUSE_ROUND_MS);
    for (uint i = 0; i < round_len; i++) {
        uint8_t k = pattern[i];
        blink(k, LED_ON_MS);
    }
}
 
static void game_over_anim(void) {
    for (int t = 0; t < 2; t++) {
        for (int i = 0; i < LED_COUNT; i++) { blink(i, 80); }
        for (int i = LED_COUNT - 1; i >= 0; i--) { blink(i, 80); }
    }
    all_blink(3, 250);
    seg_clear();
}
 
static void win_anim(void) {
    seg_show_mask(SEG_U);
    for (int t = 0; t < 6; t++) {
        int i = t % LED_COUNT;
        blink(i, 120);
    }
    all_blink(2, 300);
    seg_clear();
}
 
int main() {
    stdio_init_all();
 
    // Init LEDs
    for (uint i = 0; i < LED_COUNT; i++) {
        gpio_init(LED_PINS[i]);
        gpio_set_dir(LED_PINS[i], GPIO_OUT);
        led_off(i);
    }
    // Init botones con pull-up
    for (uint i = 0; i < BTN_COUNT; i++) {
        gpio_init(BTN_PINS[i]);
        gpio_set_dir(BTN_PINS[i], GPIO_IN);
        gpio_pull_up(BTN_PINS[i]);
    }
    // Init display
    for (int i = 0; i < DIS_COUNT; i++) {
        gpio_init(DISPLAY_PIN[i]);
        gpio_set_dir(DISPLAY_PIN[i], GPIO_OUT);
        gpio_put(DISPLAY_PIN[i], 0);
    }
 
    // Señal de encendido
    all_blink(2, 200);
    seg_show_digit(0, false);
 
    while (true) {
        // >>> Espera a pulsación para INICIAR y siembra RNG
        wait_for_start_and_seed();
 
        // Nueva partida
        round_len = 0;
        for (uint i = 0; i < MAX_STEPS; i++) pattern[i] = randi4();
 
        bool playing = true;
        while (playing) {
            if (round_len < MAX_STEPS) round_len++;
 
            // Mostrar ronda (mod 10)
            seg_show_digit(round_len, false);
 
            // Secuencia
            play_sequence();
 
            // Entrada del jugador
            for (uint i = 0; i < round_len; i++) {
                int b = wait_button_blocking();
 
                // feedback LED
                led_on(b);
                sleep_ms(INPUT_LED_MS);
                led_off(b);
 
                if ((uint8_t)b != pattern[i]) {
                    game_over_anim();
                    playing = false;
                    break;
                }
            }
 
            if (!playing) break;
 
            if (round_len >= MAX_STEPS) {
                win_anim();
                playing = false;
                break;
            }
 
            // recompensa visual
            blink(pattern[round_len - 1], 120);
        }
 
        // tras terminar, regresa a modo espera
        seg_show_digit(0, false);
        all_off();
        // el loop vuelve a wait_for_start_and_seed()
    }
    return 0;
}
 
```
## 5) Explicación del programa
a) Definiciones, temporizaciones y mapeos

Constantes de tamaño:
LED_COUNT = 4, BTN_COUNT = 4, DIS_COUNT = 8 (segmentos del display), MAX_STEPS = 15 (rondas máximas).

Tiempos (ms):
LED_ON_MS=400, LED_OFF_MS=150, INPUT_LED_MS=200, PAUSE_ROUND_MS=500, DEBOUNCE_MS=25, HOLD_MS=15.
Estos controlan velocidad del juego (dificultad) y calidad de lectura de botones.

Arreglos de pines:
LED_PINS[], BTN_PINS[], DISPLAY_PIN[] con el orden físico descrito.

b) Lógica de display 7 segmentos

Tabla DIGIT_MASKS[16]: define las máscaras bit-a-bit para 0–9 y A–F siguiendo el orden lógico A,B,C,D,E,F,G,DP (bit 0 = A, bit 7 = DP).

seg_write_mask(uint8_t m): realiza el mapeo de ese orden lógico al orden físico de pines {E,D,C,DP,G,F,A,B}.

Ejemplo: el bit de A (bit 0) termina saliendo por DISPLAY_PIN[6] (GPIO 18).

Como el display es cátodo común, 1 = encendido de segmento.

seg_show_digit(d, dp): toma d (0–15) y escribe la máscara correspondiente en el display. (El parámetro dp está previsto, pero la máscara de DP se controla dentro de seg_write_mask si quisieras extenderlo).

¿Cómo “sabe” qué número mostrar? Por la tabla de máscaras (DIGIT_MASKS) donde cada número/letra tiene definidos los segmentos que deben encenderse; luego seg_write_mask() traduce esos bits a los pines reales del display.

c) Botones y debounce

Activo en bajo: se habilita gpio_pull_up(), por lo que un botón presionado lee 0.

read_button_once(): recorre los 4 botones, aplica debounce (DEBOUNCE_MS), fuerza un tiempo mínimo de pulsación (HOLD_MS) y espera a la liberación (evita repeticiones). Devuelve el índice del botón (0–3) o -1.

wait_button_blocking(): bloquea hasta detectar una pulsación válida.

d) Aleatoriedad y arranque

wait_for_start_and_seed(): muestra 0 en el display y hace una animación cíclica de LEDs para indicar “listo para empezar”.

Cuando detecta la primera pulsación, siembra el RNG con: time_us_64() mezclado con el índice del botón; esto asegura secuencias distintas en cada partida.

Hace una pequeña pausa para que esa pulsación no cuente como entrada de juego.

e) Juego por rondas

pattern[MAX_STEPS]: secuencia aleatoria sobre {0,1,2,3}.

Flujo por partida (en main()):

Espera inicio y siembra RNG.

Inicializa round_len = 0 y llena pattern con valores aleatorios (randi4()).

Bucle de juego:

Incrementa round_len (hasta MAX_STEPS).

Muestra número de ronda en el 7-seg (0–9 y luego A–F).

Reproduce la secuencia con play_sequence() (LEDs con blink).

Entrada del jugador: por cada paso, espera botón con wait_button_blocking(), da feedback encendiendo el LED correspondiente (INPUT_LED_MS) y compara con pattern[i].

Si hay error → game_over_anim(), termina la partida.

Si completa MAX_STEPS → win_anim(), victoria.

Bonus visual: parpadeo del último elemento correcto.

Al terminar, vuelve a modo espera (display en 0, LEDs apagados).

win_anim(): muestra letra U (máscara SEG_U) como “win”, secuencia de parpadeos y limpia display.

f) Animaciones

game_over_anim(): barridos y blinks de todos los LEDs, limpia display.
### Video
[Video Pong](https://www.youtube.com/watch?v=2d4McjVFlx8&t=3s)