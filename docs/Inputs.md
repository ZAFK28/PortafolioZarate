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
5.1 Compuertas básicas AND / OR / XOR con 2 botones

Entradas: Botones A y B, configurados con pull-up (reposo=1, presionado=0).

Salidas: Tres LEDs que muestran en tiempo real el resultado de las operaciones:

LED1 = A AND B

LED2 = A OR B

LED3 = A XOR B

Demostración en video: Se deben probar las 4 combinaciones de entradas: (00, 01, 10, 11).

5.2 Selector cíclico de 4 LEDs con avance/retroceso

Entradas:

Botón AVANZA → incrementa el índice (0 → 1 → 2 → 3 → 0).

Botón RETROCEDE → decrementa el índice (0 → 3 → 2 → 1 → 0).

Salida: Solo un LED encendido a la vez entre LED0..LED3.

Condición: Cada pulsación cuenta solo una vez (antirrebote por flanco). Mantener presionado no genera múltiples avances.

Demostración en video: Se recorre en ambos sentidos mostrando el funcionamiento correcto.