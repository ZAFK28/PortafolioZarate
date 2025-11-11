# Control de Servomotor con Potenciómetro (Raspberry Pi Pico)

> Este proyecto implementa el control **analógico** de un **servomotor** mediante un **potenciómetro**, utilizando el ADC y PWM del microcontrolador **Raspberry Pi Pico**.  
> Se emplea un **filtro de media móvil** para suavizar la señal del potenciómetro y evitar movimientos bruscos del servo.

---

## 1) Resumen

- **Nombre del proyecto:** _Control de Servomotor por Potenciómetro_  
- **Autor:** _Rodrigo Zárate_  
- **Asignatura:** _Sistemas Embebidos_  
- **Fecha:** _11/11/2025_  
- **Descripción breve:** Sistema de control analógico que ajusta el ángulo de un servomotor proporcionalmente a la lectura de un potenciómetro. El valor leído se filtra mediante un promedio móvil antes de generar la señal PWM de control.

!!! tip "Información del proyecto:"
    **Lenguaje/SDK:** C++ con Raspberry Pi Pico SDK (`pico/stdlib.h`, `hardware/adc.h`, `hardware/pwm.h`)  
    **Técnicas clave:** Lectura ADC, PWM, control proporcional, filtrado con media móvil.  
    **Plataforma:** Raspberry Pi Pico / Pico 2  

### Material utilizado
- 1 × Raspberry Pi Pico o Pico 2  
- 1 × Servomotor (SG90 o similar)  
- 1 × Potenciómetro de 10 kΩ  
- Cables jumper y protoboard  

---

## 2) Objetivos

- Configurar el **ADC** para leer el valor del potenciómetro.  
- Implementar un **promedio móvil** para reducir el ruido en la señal.  
- Generar una señal **PWM** de 50 Hz para controlar la posición del servomotor.  
- Convertir la lectura del ADC (0–4095) en un ángulo de 0° a 180°.  
- Calibrar la señal PWM para producir pulsos entre 1 ms y 2 ms según el ángulo.  

---

## 3) Conexiones / Esquema

| Componente | GPIO | Descripción |
|-------------|------|-------------|
| Potenciómetro (salida central) | 26 (ADC0) | Entrada analógica |
| Servomotor (señal) | 0 | Salida PWM |
| Alimentación servo | 5V (Vsys) | Energía para el motor |
| GND común | — | Tierra compartida |

**Notas de conexión:**
- Conectar el potenciómetro como **divisor de voltaje** entre 3.3V y GND.  
- La terminal central se conecta a **GPIO26 (ADC0)**.  
- El servomotor se conecta al **GPIO0 (PWM)**.  
- Asegurarse de compartir **GND** entre el servo y la Raspberry Pi Pico.  

**Esquema básico:**


---

## 4) Código principal

```cpp
#include <stdio.h>
#include "pico/stdlib.h"
#include "hardware/adc.h"
#include "hardware/pwm.h"

#define ADC_INPUT 0
#define N_muestras 16
#define SERVO_PIN 0

uint16_t angle_to_pwm(float angle) {
    float pulse_ms = 1.0f + (angle / 180.0f);     // 1.0 → 2.0 ms
    float duty = (pulse_ms / 20.0f) * 39062.0f;   // 20 ms periodo total
    return (uint16_t)duty;
}

int main() {
    stdio_init_all();
    adc_init();
    adc_gpio_init(26);        // GPIO26 = ADC0
    adc_select_input(ADC_INPUT);

    uint16_t buffer[N_muestras];
    int sum = 0;
    uint8_t indice = 0;
    uint8_t cuenta = 0;

    gpio_set_function(SERVO_PIN, GPIO_FUNC_PWM);
    uint slice = pwm_gpio_to_slice_num(SERVO_PIN);
    pwm_set_clkdiv(slice, 64.0f);
    pwm_set_wrap(slice, 39062);
    pwm_set_enabled(slice, true);

    while (true) {
        uint16_t adc = adc_read();

        if (cuenta < N_muestras) {
            buffer[indice] = adc;
            sum += adc;
            cuenta++;
            indice++;
        } else {
            sum -= buffer[indice];
            buffer[indice] = adc;
            sum += adc;

            indice++;
            if (indice >= N_muestras) indice = 0;

            printf("Muestra:%u\n", adc);
            printf("Suma:%u\n", sum);
            printf("N_muestras:%u\n", N_muestras);

            int promedio = sum / N_muestras;
            int porcentaje = ((promedio * 180.0f) / 4200.0f);
            porcentaje = porcentaje % 180;

            printf("Promedio:%u\n", promedio);
            printf("Porcentaje:%u\n", porcentaje);

            if (porcentaje == 0) porcentaje = 1;
            pwm_set_gpio_level(SERVO_PIN, angle_to_pwm(porcentaje));
        }
    }
}
```
## 5) Explicación del código

### a) Lectura del ADC
- El **potenciómetro** se conecta al pin **GPIO26 (ADC0)**.  
- El ADC trabaja con una resolución de **12 bits**, entregando valores de `0` a `4095`.  
- Se inicializa con `adc_init()`, se asigna el pin con `adc_gpio_init(26)` y se selecciona el canal con `adc_select_input(ADC_INPUT)`.

### b) Filtro de media móvil
- Se usa un **buffer circular de 16 muestras** (`N_muestras = 16`) para suavizar la lectura del potenciómetro.  
- Cada vez que se obtiene una nueva lectura, se reemplaza la más antigua en el buffer y se recalcula la suma total.  
- El promedio (`promedio = sum / N_muestras`) reduce el ruido en la señal, evitando movimientos bruscos en el servo.

### c) Conversión del valor ADC a ángulo
- El valor promedio se convierte a un **ángulo entre 0° y 180°** con la fórmula:  
  \[
  \text{ángulo} = \frac{\text{promedio} \times 180}{4200}
  \]
- Esto garantiza una relación proporcional entre el giro del potenciómetro y la posición del servomotor.  
- Si el valor calculado es 0, se corrige a 1° para evitar pulsos nulos.

### d) Generación de señal PWM
- Se configura el **PWM** en el pin **GPIO0 (SERVO_PIN)**.  
- El ciclo de trabajo se ajusta con `pwm_set_gpio_level()` según el ángulo calculado.  
- La función `angle_to_pwm()` convierte el ángulo a una señal de pulso entre **1 ms (0°)** y **2 ms (180°)** dentro de un periodo total de **20 ms (50 Hz)**.  
- Se utilizan los parámetros:
  - `pwm_set_clkdiv(slice, 64.0f)` para dividir la frecuencia base.  
  - `pwm_set_wrap(slice, 39062)` para definir el periodo del pulso.  

### e) Flujo general del programa
1. Se leen continuamente los valores del potenciómetro.  
2. Se filtran mediante la media móvil.  
3. Se calcula el ángulo correspondiente.  
4. Se ajusta el ancho del pulso PWM, moviendo el servo de forma proporcional y estable.  
5. Se muestran en consola los valores de muestra, suma, promedio y ángulo.

---

## 6) Resultados esperados

- Al girar el **potenciómetro**, el servomotor rota suavemente de **0° a 180°**.  
- El sistema responde de forma estable, sin vibraciones gracias al filtro promedio.  
- En consola se muestran los valores de cada ciclo, facilitando la calibración y depuración.

**Ejemplo de salida en terminal:**
```
Muestra: 2100
Suma: 33600
N_muestras: 16
Promedio: 2100
Porcentaje: 90
```
**Resultado esperado:** control suave y proporcional del servomotor basado en la posición del potenciómetro, demostrando una lectura analógica estable y conversión eficiente mediante PWM.
