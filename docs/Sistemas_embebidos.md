# Sistemas embebidos

---

## Comparacion de sistemas embebidos para el desarrollo de una consola Gameboy:
### 1. ESP32 / ESP32-S3

Precio: muy barato (5–15 USD).

Pros:

Bajo consumo, perfecto para portátil con batería.

Soporta emulación NES, Game Boy, Game Boy Color, Master System sin problemas.

Muchísimos proyectos DIY en GitHub (ej. Odroid-Go, ESP32 Game Boy clones).

Contras:

No suficiente potencia para SNES fluido o PS1.

Ideal si quieres algo muy simple estilo Game Boy Classic o NES portátil.

### 2. Odroid-Go Advance (basado en Rockchip RK3326)

Precio: unos 60–80 USD con kit completo.

Pros:

Mucho más potente que un ESP32.

Corre SNES, GBA, PS1 e incluso algo de N64 y PSP ligero.

Pantalla, botones y batería ya integrados (como una consola portátil DIY).

Contras:

No tan barato como un ESP32.

Menos comunidad que Raspberry.

Ideal si quieres calidad/precio para algo ya más completo.

### 3. Anbernic / Powkiddy (consolas chinas retro listas)

Usan SoCs tipo Allwinner, Rockchip o Ingenic.
Precio: 50–120 USD.
- Pros:
    * Ya vienen listas (pantalla, batería, controles).
    * Emulan de NES hasta PS1, Dreamcast y PSP según modelo.
    * Calidad/precio insuperable si no quieres “soldar y programar”.
Contras: No es DIY (menos divertido para fabricar uno mismo).
Ejemplos buenos: RG35XX Plus, Miyoo Mini Plus, RG353V.

### 4. Compute Modules / SBC alternativos

Banana Pi, Orange Pi, Rock Pi, Radxa.

Pros: similares a Raspberry Pi, a veces más potentes y más baratos.

Contras: menos documentación y comunidad.

Útiles si quieres un “mini-PC” emulador sin pagar sobreprecio de Raspberry.

### Tabla Comparativa
| Plataforma / Chip                         | CPU                      | Velocidad (GHz)  | RAM                                            | Almacenamiento | Rendimiento en emulación                             |
| ----------------------------------------- | ------------------------ | ---------------- | ---------------------------------------------- | -------------- | ---------------------------------------------------- |
| **ESP32-S3**                              | 2× Xtensa LX7            | 240 MHz          | 512 KB SRAM interna + hasta 8 MB PSRAM externa | MicroSD        | NES, GB, GBC, SMS muy bien; SNES limitado            |
| **Raspberry Pi Zero 2 W**                 | Quad-core ARM Cortex-A53 | 1.0 GHz          | 512 MB LPDDR2                                  | MicroSD        | NES, GB, SNES, GBA bien; PS1 justo                   |
| **Raspberry Pi 3B+**                      | Quad-core ARM Cortex-A53 | 1.4 GHz          | 1 GB LPDDR2                                    | MicroSD        | SNES perfecto, GBA bien, PS1 fluido                  |
| **Raspberry Pi 4B**                       | Quad-core ARM Cortex-A72 | 1.5–1.8 GHz (OC) | 2–8 GB LPDDR4                                  | MicroSD / SSD  | SNES, GBA, PS1 perfecto; N64 y PSP jugable (no todo) |
| **Odroid-Go Advance (RK3326)**            | Quad-core ARM Cortex-A35 | 1.3 GHz          | 1 GB LPDDR3                                    | MicroSD        | NES→PS1 muy bien; N64 y PSP con límites              |
| **Anbernic RG35XX Plus (Allwinner H700)** | Quad-core Cortex-A53     | 1.5 GHz          | 1 GB LPDDR4                                    | MicroSD        | NES→PS1 perfecto; GBA excelente; N64 parcial         |
