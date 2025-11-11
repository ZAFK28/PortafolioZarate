# Medición de Iluminancia (Luxómetro) con ADC en Raspberry Pi Pico

> Este proyecto implementa un **luxómetro digital** utilizando el convertidor analógico–digital (ADC) del microcontrolador **Raspberry Pi Pico**.  
> El sistema realiza una **lectura continua de luz ambiental**, aplica una **media móvil** sobre 16 muestras y calcula un **porcentaje de luminosidad** aproximado.

---

## 1) Resumen

- **Nombre del proyecto:** _Luxómetro con ADC y media móvil_  
- **Autor:** _Rodrigo Zárate_  
- **Asignatura:** _Sistemas Embebidos_  
- **Fecha:** _11/11/2025_  
- **Descripción breve:** Sistema que mide el nivel de iluminación mediante un sensor analógico conectado al pin ADC0 (GPIO26), filtrando el ruido con un promedio móvil y mostrando el porcentaje de luz en consola.

!!! tip "Información del proyecto:"
    **Lenguaje/SDK:** C con Raspberry Pi Pico SDK (`pico/stdlib.h`, `hardware/adc.h`)  
    **Técnicas clave:** Lectura ADC, media móvil, procesamiento de señales analógicas.  
    **Plataforma:** Raspberry Pi Pico / Pico 2  

### Material utilizado
- 1 × Raspberry Pi Pico o Pico 2  
- 1 × Sensor de luz (LDR o fototransistor)  
- 1 × Resistencia fija (10 kΩ) para divisor de tensión  
- Cables jumper y protoboard  

---

## 2) Objetivos

- Configurar el **ADC** del Pico para leer niveles de voltaje del sensor LDR.  
- Implementar un **filtro de media móvil** con 16 muestras.  
- Calcular un **porcentaje de luminosidad relativa** basado en las lecturas promedio.  
- Mostrar en consola los valores instantáneos, la suma de muestras y el resultado filtrado.  

---

## 3) Conexiones / Esquema

| Componente | GPIO | Descripción |
|-------------|------|-------------|
| LDR (sensor de luz) | 26 (ADC0) | Entrada analógica |
| Resistencia fija (10 kΩ) | — | Conectada en divisor de tensión |
| GND | — | Tierra común |

**Notas de conexión:**
- El LDR se conecta en un divisor de voltaje con una resistencia de 10 kΩ.  
- La unión central del divisor se conecta a **GPIO26 (ADC0)**.  
- Conectar **GND común** y alimentación de **3.3 V** al sensor.  

**Esquema simplificado:**


---

## 4) Código principal

```cpp
#include <stdio.h>
#include "pico/stdlib.h"
#include "hardware/adc.h"

#define ADC_INPUT 0       // Canal ADC0
#define N_muestras 16     // Número de muestras para el promedio

int main() {
    stdio_init_all();
    adc_init();
    adc_gpio_init(26);     // GPIO26 = ADC0
    adc_select_input(ADC_INPUT);

    uint16_t buffer[N_muestras];
    int sum = 0;
    uint8_t indice = 0;
    uint8_t cuenta = 0;

    while (true) {
        uint16_t adc = adc_read(); // Lectura ADC (0–4095)

        if (cuenta < N_muestras) {
            buffer[indice] = adc;
            sum += adc;
            cuenta++;
            indice++;
        } else {
            sum -= buffer[indice];       // Restar valor más antiguo
            buffer[indice] = adc;        // Agregar nuevo valor
            sum += adc;                  // Sumar nuevo valor
            indice++;
            if (indice >= N_muestras) indice = 0;

            printf("Muestra:%u\n", adc);
            printf("Suma:%u\n", sum);
            printf("N_muestras:%u\n", N_muestras);

            int promedio = sum / N_muestras;
            int porcentaje = ((-promedio * 100.0f) / 2300.0f) + 100;
            porcentaje = porcentaje % 100;

            printf("Promedio:%u\n", promedio);
            printf("Porcentaje:%u%%\n", porcentaje);
            sleep_ms(200);
        }
    }
}
```
## 5) Explicación del código

### a) Lectura del ADC
- Se inicializa el ADC interno de 12 bits de resolución (0–4095).  
- `adc_gpio_init(26)` configura el pin GPIO26 como entrada analógica.  
- `adc_select_input(0)` selecciona el canal ADC0 correspondiente.

### b) Promedio móvil
- Se almacenan **16 muestras** en un buffer circular.  
- Cada nueva lectura reemplaza la más antigua, actualizando la suma total.  
- Se obtiene el promedio dividiendo la suma entre `N_muestras`.

### c) Cálculo del porcentaje
- El promedio se transforma en un valor porcentual relativo a un rango máximo (~2300).  
- Este valor puede calibrarse según las condiciones de luz ambiental o el tipo de sensor.

### d) Salida en consola
- Se imprimen valores en consola: muestra individual, suma acumulada, promedio y porcentaje de luz.  
- `sleep_ms(200)` reduce la frecuencia de muestreo a ~5 Hz para mejor lectura humana.

---

## 6) Resultados esperados

- En entornos **oscuros**, el valor ADC aumenta → porcentaje bajo.  
- En **luz intensa**, el ADC disminuye → porcentaje alto.  
- Los valores se actualizan de forma continua y estable, filtrados con la media móvil.  

**Ejemplo de salida en terminal:**
```
Muestra: 1800
Suma: 28800
N_muestras: 16
Promedio: 1800
Porcentaje: 21%
```
**Resultado esperado:** medición estable de iluminación ambiental con respuesta proporcional a la cantidad de luz recibida por el sensor.
