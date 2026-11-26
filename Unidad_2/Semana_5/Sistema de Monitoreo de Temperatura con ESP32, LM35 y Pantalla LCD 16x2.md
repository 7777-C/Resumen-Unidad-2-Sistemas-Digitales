# Resumen: Procesamiento Digital de Señales y Pantallas LCD 16x2 con ESP32

## 1. Fundamentos del Procesamiento Digital de Señales (DSP)

El entorno físico es **analógico**, mientras que un microcontrolador es **digital**. El DSP consiste en convertir esa señal continua en una secuencia de números para escalarlos, limpiarlos, tomar decisiones y mostrarlos.

**Cadena de adquisición y procesamiento:**
Sensor (LM35: °C a mV) → ADC (Muestrea y cuantiza) → Código (Escala, filtra y decide) → Salida (Pantalla LCD / Alarma)

### Muestreo (¿Cada cuánto tiempo medimos?)
* **Muestra (`x[n]`) y Muestreo:** Una muestra es el valor de la señal en un instante preciso; el muestreo es la captura periódica de estas muestras a intervalos fijos.
* **Periodo (`Ts`) y Frecuencia (`fs`):** `Ts` es el tiempo en segundos entre dos muestras consecutivas. La frecuencia `fs` es la cantidad de muestras tomadas por segundo (`fs = 1 / Ts`, en Hz). Por ejemplo, un `Ts` de 100 ms (0.1 s) equivale a una `fs` de 10 Hz.
* **Teorema de Nyquist y Aliasing:** Para reconstruir una señal sin distorsión, la frecuencia de muestreo debe ser mayor al doble de la frecuencia máxima de la señal (`fs > 2 * f_max`). Si se muestrea demasiado lento ocurre el **aliasing**, generando una señal falsa de menor frecuencia (`f_falsa = |f - k * fs|`).
* **Aplicación a la temperatura:** Dado que la temperatura varía muy despacio (< 1 Hz), muestrear a 10 Hz cumple ampliamente con Nyquist y permite promediar lecturas sin desperdiciar energía ni capacidad de procesamiento.

### Cuantización (¿Con qué detalle medimos cada muestra?)
* **Concepto y Error:** Cuantizar es redondear cada muestra al escalón digital más cercano. La diferencia perdida en el redondeo se conoce como **error de cuantización** (como máximo ±0.5 escalones).
* **Resolución y LSB:** Con N bits se obtienen `2^N` escalones posibles. El **LSB** (Bit Menos Significativo) es el valor de voltaje de un solo escalón (`LSB = Voltaje de referencia / 2^N`).
  * **Arduino UNO (10 bits, 5 V):** 1024 escalones → 5000 mV / 1024 ≈ 4.9 mV por escalón.
  * **ESP32 (12 bits, 1.1 V base):** 4096 escalones → 1100 mV / 4096 ≈ 0.27 mV por escalón.
* **Resolución vs. Exactitud:** Tener escalones muy pequeños (alta resolución) no garantiza que la lectura coincida con el valor real (exactitud) si existe ruido eléctrico o descalibración.
* **Funcionamiento del ADC (SAR):** El conversor del ESP32 realiza una búsqueda binaria haciendo 12 comparaciones consecutivas (una por cada bit) hasta determinar el número digital correspondiente al voltaje de entrada.

---

## 2. Hardware: ADC del ESP32 y Sensor LM35

### Configuración del ADC en el ESP32
* **Selección de pin:** Se debe utilizar el módulo **ADC1** (pines GPIO 32 al 39, como el GPIO34), ya que el módulo ADC2 queda inhabilitado cuando se activa el Wi-Fi.
* **Atenuación:** El ADC del ESP32 tiene un rango nativo de 0 a 1.1 V. Para medir voltajes mayores utiliza divisores internos llamados atenuación. Se debe escoger la atenuación más baja que cubra el voltaje máximo del sensor para no perder resolución:
  * `ADC_0db` (100 a 950 mV): Recomendada para el LM35 en ambientes interiores (10 a 95 °C). La atenuación por defecto (`11db`, 150 a 3100 mV) lee mal temperaturas bajas por debajo de 15 °C.
* **Calibración y Ruido:** Se usa `analogReadMilliVolts(pin)` en lugar de `analogRead()` para aprovechar los datos de calibración grabados de fábrica en el chip (`eFuse`). Para filtrar ruido eléctrico se coloca un **condensador cerámico de 100 nF ("104")** entre el pin del sensor y GND.

### El sensor LM35
* **Funcionamiento:** Entrega 10 mV por cada grado Celsius (`T = mV / 10`).
* **Conexión:** Se alimenta con **VIN (5 V)** (con 3.3 V falla). Su salida máxima es 1.5 V a 150 °C, por lo que no daña el pin del ESP32.

---

## 3. Procesamiento Digital por Software

1. **Escalamiento:** Dividir los milivoltios entre `10.0` (en punto flotante `float` para no perder decimales).
2. **Filtrado (Media Móvil):** Promediar las últimas N muestras (N = 10 a 10 Hz equivale al último segundo) para suavizar el ruido. Se implementa con un **búfer circular** (el dato nuevo sobrescribe al más viejo). Introduce un retardo de respuesta de `N * Ts` (1 s).
3. **Decisión con Histéresis:** Evita el parpadeo errático de una alarma por ruido usando **dos umbrales** distintos (ej. encender a 28 °C y apagar al bajar a 26 °C) en lugar de uno solo.

---

## 4. Visualización en Pantalla LCD 16x2

### Arquitectura Interna
* **Estructura:** 16 columnas (`0` a `15`) y 2 filas (`0` y `1`), sumando 32 casillas. Cada casilla es una matriz de 5 x 8 puntos (la última fila es para el cursor).
* **Controlador HD44780:** Recibe códigos **ASCII** (ej. `65` para `'A'`) desde el ESP32 y gestiona tres memorias:
  * **CGROM:** Formas de letras estándar grabadas de fábrica.
  * **DDRAM:** Códigos de los caracteres que se muestran en cada casilla.
  * **CGRAM:** Espacio para crear hasta **8 caracteres propios** (`0` a `7`), como el símbolo de grados (`°`), usando `createChar()` y mostrándolo con `write()`.

### Conexión Directa vs. Módulo I2C
Para comunicarse con la LCD se usan pines de control: **RS** (`0` = comando, `1` = dato), **R/W** (a GND para solo escribir) y **E** (*Enable*, pulso que ordena leer los datos en su flanco de bajada).

| Característica | Conexión Directa (Modo 4 bits) | Módulo I2C (Chip PCF8574) |
| :--- | :--- | :--- |
| **Pines del ESP32** | 6 pines (RS, E, D4 a D7) | 2 pines (SDA en GPIO21, SCL en GPIO22) |
| **Cableado total** | ~12 cables + potenciómetro externo | 4 cables (GND, VCC a 5V, SDA, SCL) |
| **Contraste** | Potenciómetro externo | Tornillo azul integrado en el módulo |
| **Dirección I2C** | No aplica | `0x27` o `0x3F` (verificable con escáner I2C) |

*Nota eléctrica:* Aunque el ESP32 opera a 3.3 V y el módulo I2C a 5 V, la corriente de sus resistencias pull-up es mínima (~0.2 mA), haciendo segura la conexión directa.

### Programación y Buenas Prácticas
Se utiliza la librería `LiquidCrystal_I2C` con funciones como `lcd.init()`, `lcd.backlight()`, `lcd.setCursor(col, fila)` y `lcd.print()`.

* **Separar fijo de variable:** Escribir etiquetas fijas (ej. `"Temp:"`) una sola vez en `setup()`, y actualizar únicamente los datos cambiantes dentro de `loop()` posicionando el cursor.
* **Evitar `lcd.clear()` en el ciclo:** Es lento (~2 ms) y provoca parpadeo. Para evitar que queden dígitos residuales (ej. pasar de `"25.10"` a `"9.5"` y ver `"9.510"`), se sobrescribe el valor rellenando con espacios en blanco.
* **Ahorro energético:** La luz de fondo es el componente de mayor consumo; conviene apagarla con `lcd.noBacklight()` y activarla solo ante una alarma o interacción del usuario.
