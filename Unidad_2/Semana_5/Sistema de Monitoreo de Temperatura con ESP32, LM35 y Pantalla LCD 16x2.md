# Sistema de Monitoreo de Temperatura con ESP32, LM35 y Pantalla LCD 16x2

El mundo físico es continuo y los microcontroladores son **digitales**. Para procesar variables físicas en un ESP32 se sigue esta cadena:

$$
\text{Sensor (LM35: °C} \rightarrow \text{mV)} \longrightarrow \text{ADC (Muestrea y cuantiza)} \longrightarrow \text{Código (Escala, filtra, decide)} \longrightarrow \text{Salida (LCD 16x2)}
$$

## 1. Digitalización de la Señal: Muestreo y Cuantización

### Muestreo (¿Cada cuánto tiempo medimos?)
* **Muestra ($x[n]$) y Muestreo:** Capturar "fotos" de la señal a intervalos fijos de tiempo, como los fotogramas de una película.
* **Periodo ($T_s$) y Frecuencia ($f_s$):** $T_s$ es el tiempo en segundos entre cada muestra y $f_s = 1 / T_s$ es la cantidad de muestras por segundo (Hz). Si $T_s = 100\text{ ms}$ ($0.1\text{ s}$), entonces $f_s = 10\text{ Hz}$.
* **Teorema de Nyquist y Aliasing:** La frecuencia de muestreo debe ser mayor al doble de la frecuencia máxima de la señal ($f_s > 2 \cdot f_{\max}$). Muestrear muy lento provoca **aliasing**, generando una señal falsa de baja frecuencia ($f_{\text{falsa}} = \vert f - k \cdot f_s \vert$), similar al efecto de ruedas girando al revés en un video.
* **Temperatura:** Varía muy lento ($< 1\text{ Hz}$). Muestrear a $10\text{ Hz}$ cumple con Nyquist, permite promediar y ahorra CPU y energía.

### Cuantización (¿Con qué detalle medimos cada muestra?)
* **Cuantización y Error:** Es redondear cada muestra al escalón digital más cercano. Lo que se pierde al redondear es el **error de cuantización** (máximo $\pm \frac{1}{2}$ escalón).
* **Resolución y LSB:** Con $N$ bits existen $2^N$ escalones. El **LSB** (Bit Menos Significativo) es el voltaje de un escalón ($\text{LSB} = \text{Fondo de escala} / 2^N$).
  * *Arduino UNO (10 bits, $5\text{ V}$):* $1024$ escalones $\rightarrow 5000\text{ mV} / 1024 \approx 4.9\text{ mV}$ por escalón.
  * *ESP32 (12 bits, $1.1\text{ V}$ base):* $4096$ escalones (`0` a `4095`) $\rightarrow 1100\text{ mV} / 4096 \approx 0.27\text{ mV}$ por escalón.
* **ADC tipo SAR:** Encuentra el valor haciendo una búsqueda binaria de 12 preguntas consecutivas (una por bit).
* **Resolución vs. Exactitud:** Tener escalones pequeños (alta resolución) no garantiza cercanía al valor real (exactitud) si hay ruido o descalibración.

## 2. Configuración del Hardware: ESP32 y Sensor LM35

### El ADC del ESP32
* **Pines:** Se usa el **ADC1** (GPIO 32 al 39, específicamente **GPIO34**), porque el ADC2 deja de funcionar al activar el Wi-Fi.
* **Atenuación ("El zoom del ADC"):** La regla interna mide hasta $\approx 1.1\text{ V}$. Para rangos mayores usa divisores internos llamados atenuación. Se elige la atenuación más baja que cubra el voltaje máximo del sensor:
  * **`ADC_0db` ($100\text{ a } 950\text{ mV}$):** Recomendada para el LM35 en un aula ($10\text{ a } 95\text{ °C}$). La atenuación por defecto (`11db`, $150\text{ a } 3100\text{ mV}$) lee mal por debajo de $15\text{ °C}$.
* **Calibración y Ruido:** Se usa `analogReadMilliVolts(pin)` en lugar de `analogRead()` para aprovechar los datos de calibración grabados de fábrica en el chip (*eFuse*). Para filtrar ruido eléctrico se coloca un **condensador cerámico de 100 nF ("104")** entre el pin del sensor y GND.

### El Sensor LM35
* **Funcionamiento:** Entrega **$10\text{ mV}$ por cada grado Celsius** ($T = \text{mV} \div 10$).
* **Conexión:** Se alimenta con **VIN ($5\text{ V}$)** (con $3.3\text{ V}$ falla). Su salida máxima es $1.5\text{ V}$ a $150\text{ °C}$, por lo que no daña el pin del ESP32.

## 3. Procesamiento Digital por Software

1. **Escalamiento:** Dividir los milivoltios entre `10.0` (en punto flotante `float` para no perder decimales).
2. **Filtrado (Media Móvil):** Promediar las últimas $N$ muestras ($N=10$ a $10\text{ Hz}$ equivale al último segundo) para suavizar el ruido. Se implementa con un **búfer circular** (el dato nuevo sobrescribe al más viejo). Introduce un retardo de respuesta de $N \times T_s$ ($1\text{ s}$).
3. **Decisión con Histéresis:** Evita el parpadeo errático de una alarma por ruido usando **dos umbrales** distintos (ej. encender a $28\text{ °C}$ y apagar al bajar a $26\text{ °C}$) en lugar de uno solo.

## 4. Visualización en Pantalla LCD 16x2

### Arquitectura Interna
* **Estructura:** **16 columnas** (`0` a `15`) y **2 filas** (`0` y `1`), sumando 32 casillas. Cada casilla es una matriz de $5 \times 8$ puntos (la última fila es para el cursor).
* **Controlador HD44780:** Recibe códigos **ASCII** (ej. `65` para `'A'`) desde el ESP32 y gestiona tres memorias:
  * **CGROM:** Formas de letras estándar grabadas de fábrica.
  * **DDRAM:** Códigos de los caracteres que se muestran en cada casilla.
  * **CGRAM:** Espacio para crear hasta **8 caracteres propios** (`0` a `7`), como el símbolo de grados (`°`), usando `createChar()` y mostrándolo con `write()`.

### Conexión Directa vs. Módulo I2C
Para comunicarse con la LCD se usan pines de control (**RS** para comando `0` o dato `1`, **R/W** a GND para solo escribir, y **E** o *Enable* como botón de envío en flanco de bajada) y datos:

| Aspecto | Conexión Directa (Modo 4 bits) | Con Módulo I2C (Chip PCF8574) |
| :--- | :--- | :--- |
| **Pines del ESP32** | **6 pines** (RS, E, D4–D7) | **2 pines** (SDA en GPIO21, SCL en GPIO22) |
| **Cableado total** | $\approx 12$ cables + potenciómetro | **4 cables** (GND, VCC a 5V, SDA, SCL) |
| **Contraste** | Potenciómetro externo en pin VO | Tornillo azul integrado en el módulo |
| **Identificación** | Directa por hardware | Dirección I2C (`0x27` o `0x3F`) |

*Nota:* Aunque el módulo I2C se alimenta a $5\text{ V}$, la corriente de retorno hacia los pines de $3.3\text{ V}$ del ESP32 a través de sus resistencias *pull-up* es muy baja ($\approx 0.2\text{ mA}$), por lo que la conexión directa es segura.

### Programación y Buenas Prácticas
Con la librería `LiquidCrystal_I2C` (`lcd.init()`, `lcd.setCursor(col, fila)`, `lcd.print()`):
* **Separar lo fijo de lo variable:** Escribir textos estáticos (`"Temp:"`, `"Alarma:"`) una vez en `setup()`, y actualizar solo los datos cambiantes en `loop()`.
* **Evitar parpadeos y restos:** No usar `lcd.clear()` en `loop()` porque demora $\approx 2\text{ ms}$ y hace parpadear la pantalla. Para evitar dígitos residuales al pasar de números largos a cortos (ej. `"25.10"` a `"9.5"` $\rightarrow$ `"9.510"`), sobrescribir rellenando con espacios en blanco.
* **Sostenibilidad:** Apagar la luz de fondo (`lcd.noBacklight()`) cuando no se observa la pantalla y encenderla solo ante alarmas o pulsaciones para ahorrar energía.
