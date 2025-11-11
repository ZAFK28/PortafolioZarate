# Comunicación I2C con DS1307 (Reloj) y MPU6050 (Acelerómetro) – Raspberry Pi Pico

> Este proyecto establece comunicación **I2C doble** con dos dispositivos diferentes usando **dos puertos I2C** del Raspberry Pi Pico:  
> - Un **DS1307** (reloj de tiempo real, RTC) para leer **hora, minuto y segundo**.  
> - Un **MPU6050** para leer la **aceleración en los ejes X, Y y Z**.  

---

## 1) Resumen

- **Nombre del proyecto:** _Lectura de RTC DS1307 y acelerómetro MPU6050 por I2C_  
- **Autor:** _Rodrigo Zárate_  
- **Asignatura:** _Sistemas Embebidos_  
- **Fecha:** _11/11/2025_  
- **Descripción breve:** Uso de los dos puertos I2C del Raspberry Pi Pico para leer en paralelo la hora actual desde un DS1307 y la aceleración en los tres ejes desde un MPU6050, mostrando los datos en la consola serie.

!!! tip "Información del proyecto:"
    **Lenguaje/SDK:** C con Raspberry Pi Pico SDK (`pico/stdlib.h`, `hardware/i2c.h`)  
    **Técnicas clave:** Comunicación I2C, conversión BCD–decimal, lectura de registros, procesamiento de datos de acelerómetro.  
    **Plataforma:** Raspberry Pi Pico / Pico 2  

---

## 2) Objetivos

- Configurar **dos puertos I2C** del Raspberry Pi Pico (`i2c0` e `i2c1`).  
- Leer la **hora (hh:mm:ss)** desde el módulo de reloj **DS1307**.  
- Leer la **aceleración en X, Y y Z** desde el sensor **MPU6050**.  
- Convertir datos en formato **BCD** a decimal para el reloj.  
- Convertir datos crudos del acelerómetro a unidades de **g**.  
- Mostrar toda la información en la consola serie cada segundo.

---

## 3) Conexiones / Esquema

| Dispositivo | Puerto I2C | GPIO SDA | GPIO SCL | Dirección I2C |
|------------|-----------|----------|----------|---------------|
| DS1307 (RTC) | I2C0 | GP4 | GP5 | `0x68` |
| MPU6050 (Acelerómetro) | I2C1 | GP2 | GP3 | `0x68` |

**Notas de conexión:**
- Ambos dispositivos usan la misma dirección (`0x68`), pero están en **puertos I2C diferentes**, por eso no hay conflicto.  
- Conectar **VCC del DS1307 y MPU6050 a 3.3V o 5V** (según el módulo, muchos traen regulador).  
- Conectar **GND común** entre Pico, DS1307 y MPU6050.  
- En muchos módulos ya vienen resistencias pull-up en SDA/SCL; si no, usar resistencias de 4.7 kΩ a 3.3 V.  

---

## 4) Código principal

```cpp
#include <stdio.h>
#include "pico/stdlib.h"
#include "hardware/i2c.h"

#define I2C_PORT    i2c0       // DS1307
#define I2C_PORTA   i2c1       // MPU6050
#define SDA_PINR    4
#define SCL_PINR    5
#define SDA_PINA    2
#define SCL_PINA    3

#define ADDR  0x68   // Dirección DS1307
#define ADDRA 0x68   // Dirección MPU6050

static inline uint8_t bcd2dec(uint8_t v) {
    return (uint8_t)((v >> 4) * 10 + (v & 0x0F));
}

static inline int16_t make16(uint8_t hi, uint8_t lo) {
    return (int16_t)((hi << 8) | lo);
}

int main(void) {
    stdio_init_all();
    
    i2c_init(I2C_PORT, 100000);
    i2c_init(I2C_PORTA, 100000);

    gpio_set_function(SDA_PINR, GPIO_FUNC_I2C);
    gpio_set_function(SCL_PINR, GPIO_FUNC_I2C);
    gpio_set_function(SDA_PINA, GPIO_FUNC_I2C);
    gpio_set_function(SCL_PINA, GPIO_FUNC_I2C);

    gpio_pull_up(SDA_PINR);
    gpio_pull_up(SCL_PINR);
    gpio_pull_up(SDA_PINA);
    gpio_pull_up(SCL_PINA);

    sleep_ms(500);

    while (true) {
        // ---------- Lectura del DS1307 (Reloj) ----------
        uint8_t regR = 0x00;     // Registro de segundos
        uint8_t memoriaR[3];     // seg, min, hour

        int wR = i2c_write_blocking(I2C_PORT, ADDR, &regR, 1, true);
        if (wR < 0) {
            printf("I2C R error (ret=%d)\n", wR);
        } else if (wR != 1) {
            printf("R Escritura parcial: %d/1 bytes\n", wR);
        } else {
            int rR = i2c_read_blocking(I2C_PORT, ADDR, memoriaR, sizeof(memoriaR), false);
            if (rR < 0) {
                printf("I2C R error (ret=%d)\n", rR);
            } else if (rR != (int)sizeof(memoriaR)) {
                printf("R Lectura parcial: %d/%d bytes\n", rR, (int)sizeof(memoriaR));
            } else {
                uint8_t seg_bcd  = memoriaR[0];
                uint8_t min_bcd  = memoriaR[1];
                uint8_t hour_raw = memoriaR[2];

                uint8_t seg  = bcd2dec(seg_bcd);
                uint8_t min  = bcd2dec(min_bcd);
                uint8_t hour = bcd2dec(hour_raw);

                printf("Reloj = %02u:%02u:%02u\n", hour, min, seg);
            }
        }

        // ---------- Lectura del MPU6050 (Acelerómetro) ----------
        uint8_t regA = 0x3B;     // Primer registro de aceleración
        uint8_t memoriaA[6];     // AX, AY, AZ (2 bytes cada uno)

        int wA = i2c_write_blocking(I2C_PORTA, ADDRA, &regA, 1, true);
        if (wA < 0) {
            printf("I2C A error (ret=%d)\n", wA);
        } else if (wA != 1) {
            printf("A Escritura parcial: %d/1 bytes\n", wA);
        } else {
            int rA = i2c_read_blocking(I2C_PORTA, ADDRA, memoriaA, sizeof(memoriaA), false);
            if (rA < 0) {
                printf("I2C A error (ret=%d)\n", rA);
            } else if (rA != (int)sizeof(memoriaA)) {
                printf("A Lectura parcial: %d/%d bytes\n", rA, (int)sizeof(memoriaA));
            } else {
                int16_t ax_raw = make16(memoriaA[0], memoriaA[1]);
                int16_t ay_raw = make16(memoriaA[2], memoriaA[3]);
                int16_t az_raw = make16(memoriaA[4], memoriaA[5]);

                const float scale = 16384.0f; // ±2g
                float ax_g = (float)ax_raw / scale;
                float ay_g = (float)ay_raw / scale;
                float az_g = (float)az_raw / scale;

                printf("Aceleracion = X: %.3fg  Y: %.3fg  Z: %.3fg\n",
                       ax_g, ay_g, az_g);
            }
        }

        sleep_ms(1000);
    }

    return 0;
}
```
## 5) Explicación del código

### a) Configuración de los puertos I2C
- Se utilizan **dos interfaces I2C** del Raspberry Pi Pico:
  - `i2c0` para el **DS1307** (reloj de tiempo real).
  - `i2c1` para el **MPU6050** (acelerómetro).
- Ambas se inicializan con una frecuencia de **100 kHz** usando `i2c_init()`.
- Los pines asignados son:
  - **I2C0:** SDA en GP4, SCL en GP5.
  - **I2C1:** SDA en GP2, SCL en GP3.
- Cada línea I2C cuenta con resistencias de **pull-up** activadas con `gpio_pull_up()`.

### b) Conversión de formatos
- El reloj **DS1307** entrega los valores en formato **BCD** (Binary-Coded Decimal).
- La función auxiliar `bcd2dec()` convierte el valor a decimal dividiendo las decenas y unidades.
- Para el **MPU6050**, los valores de aceleración vienen en dos bytes (alto y bajo).  
  La función `make16()` une ambos para formar un entero de 16 bits con signo.

### c) Lectura del DS1307 (Reloj)
1. Se selecciona el registro inicial `0x00` para comenzar a leer desde los segundos.  
2. Se leen tres bytes correspondientes a segundos, minutos y horas.  
3. Cada valor se convierte de BCD a decimal mediante `bcd2dec()`.  
4. Finalmente se imprime la hora en el formato `hh:mm:ss`:
   ```cpp
   printf("Reloj = %02u:%02u:%02u\n", hour, min, seg);
   ```

### d) Lectura del MPU6050 (Acelerómetro)
1. Se selecciona el registro `0x3B`, donde comienzan los datos de aceleración.  
2. Se leen **6 bytes**: dos para cada eje (X, Y, Z).  
3. Los valores se convierten con `make16()` y se escalan a unidades de **g** usando `1g = 16384 LSB`.  
4. Los datos se muestran así:
   ```cpp
   printf("Aceleracion = X: %.3fg  Y: %.3fg  Z: %.3fg\n", ax_g, ay_g, az_g);
   ```

### e) Bucle principal
- Dentro de `while(true)`, se ejecutan las lecturas del reloj y el acelerómetro continuamente.  
- Cada ciclo espera **1 segundo** con `sleep_ms(1000)` para actualizar los valores.  
- Si ocurre un error I2C, el programa lo reporta en la consola mediante mensajes de depuración.

---

## 6) Resultados esperados

- En la consola serie aparecerán líneas como las siguientes:
  ```
  Reloj = 14:25:31
  Aceleracion = X: 0.010g  Y: -0.045g  Z: 0.998g
  ```
- El reloj **DS1307** mostrará la hora actual avanzando segundo a segundo.  
- El **MPU6050** reflejará los cambios de aceleración al mover o inclinar el sensor.  
- En reposo, los valores típicos serán:
  - X ≈ 0 g  
  - Y ≈ 0 g  
  - Z ≈ 1 g  
- **Resultado esperado:** comunicación I2C estable y simultánea con dos sensores distintos, obteniendo tanto la hora como los datos de aceleración en tiempo real.
