# Sistema de Monitoreo de Temperatura con ESP32, LM35 y Pantalla LCD 16x2

El mundo físico es continuo y los microcontroladores son **digitales**. Para procesar variables físicas en un ESP32 se sigue esta cadena:

$$
\mathrm{Sensor\ (LM35:\ ^\circ C \rightarrow mV)}
\longrightarrow
\mathrm{ADC\ (Muestreo\ y\ cuantización)}
\longrightarrow
\mathrm{Código\ (Escala,\ filtra,\ decide)}
\longrightarrow
\mathrm{Salida\ (LCD\ 16x2)}
$$

## 1. Digitalización de la Señal: Muestreo y Cuantización

### 1.1. Muestreo: ¿Cada cuánto tiempo medimos?

- **Muestra y Muestreo:** Una muestra es una captura del valor de una señal en un instante determinado. El muestreo consiste en capturar estas «fotos» de la señal a intervalos fijos de tiempo, como los fotogramas de una película.

- **Periodo ($T_s$) y Frecuencia ($f_s$):** $T_s$ es el tiempo en segundos entre cada muestra y $f_s$ es la cantidad de muestras por segundo, medida en Hz.

  La relación entre ambas magnitudes es:

  $$
  f_s = \frac{1}{T_s}
  $$

  Si $T_s = 100\ \mathrm{ms}$ ($0.1\ \mathrm{s}$), entonces:

  $$
  f_s = \frac{1}{0.1} = 10\ \mathrm{Hz}
  $$

- **Teorema de Nyquist y Aliasing:** La frecuencia de muestreo debe ser mayor al doble de la frecuencia máxima de la señal:

  $$
  f_s > 2 \cdot f_{\max}
  $$

  Muestrear muy lento provoca **aliasing**, generando una señal falsa de baja frecuencia:

  $$
  f_{\mathrm{falsa}} = \left|f - k \cdot f_s\right|
  $$

  Esto es similar al efecto de ruedas girando al revés en un video.

- **Temperatura:** Varía muy lento ($< 1\ \mathrm{Hz}$). Muestrear a $10\ \mathrm{Hz}$ cumple con Nyquist, permite promediar y ahorrar CPU y energía.

### 1.2. Cuantización: ¿Con qué detalle medimos cada muestra?

- **Cuantización y Error:** Consiste en redondear cada muestra al escalón digital más cercano. Lo que se pierde al redondear es el **error de cuantización**, cuyo valor máximo es:

  $$
  E_{\max} = \pm \frac{1}{2}\text{ escalón}
  $$

- **Resolución y LSB:** Con $N$ bits existen $2^N$ escalones. El **LSB (Bit Menos Significativo)** representa el voltaje de un escalón:

  $$
  \mathrm{LSB} = \frac{\mathrm{Fondo\ de\ escala}}{2^N}
  $$

  - **Arduino UNO (10 bits, $5\ \mathrm{V}$):** Tiene $1024$ escalones.

    $$
    \mathrm{LSB} = \frac{5000\ \mathrm{mV}}{1024}
    \approx 4.9\ \mathrm{mV}
    $$

    Cada escalón equivale aproximadamente a $4.9\ \mathrm{mV}$.

  - **ESP32 (12 bits, $1.1\ \mathrm{V}$ base):** Tiene $4096$ niveles (`0` a `4095`).

    $$
    \mathrm{LSB} = \frac{1100\ \mathrm{mV}}{4096}
    \approx 0.27\ \mathrm{mV}
    $$

    Cada escalón equivale aproximadamente a $0.27\ \mathrm{mV}$.

- **ADC tipo SAR:** Encuentra el valor haciendo una búsqueda binaria de 12 preguntas consecutivas, una por cada bit.

- **Resolución vs. Exactitud:** Tener escalones pequeños (alta resolución) no garantiza cercanía al valor real (exactitud) si hay ruido o descalibración.

## 2. Configuración del Hardware: ESP32 y Sensor LM35

### 2.1. El ADC del ESP32

- **Pines:** Se usa el **ADC1** (GPIO 32 al 39, específicamente **GPIO34**), porque el ADC2 deja de funcionar al activar el Wi-Fi.

- **Atenuación («El zoom del ADC»):** La regla interna mide hasta aproximadamente $1.1\ \mathrm{V}$. Para rangos mayores utiliza divisores internos llamados atenuación. Se elige la atenuación más baja que cubra el voltaje máximo del sensor:

  - **`ADC_0db` ($100$ a $950\ \mathrm{mV}$):** Recomendada para el LM35 en un aula ($10$ a $95\ ^\circ\mathrm{C}$).
  - La atenuación por defecto (`11db`, $150$ a $3100\ \mathrm{mV}$) puede leer mal por debajo de $15\ ^\circ\mathrm{C}$.

- **Calibración y Ruido:** Se usa `analogReadMilliVolts(pin)` en lugar de `analogRead()` para aprovechar los datos de calibración grabados de fábrica en el chip (*eFuse*). Para filtrar ruido eléctrico se coloca un **condensador cerámico de 100 nF («104»)** entre el pin del sensor y GND.

### 2.2. El Sensor LM35

- **Funcionamiento:** Entrega **$10\ \mathrm{mV}$ por cada grado Celsius**.

  La temperatura se calcula mediante:

  $$
  T = \frac{\mathrm{mV}}{10}
  $$

- **Conexión:** Se alimenta con **VIN ($5\ \mathrm{V}$)** (con $3.3\ \mathrm{V}$ falla). Su salida máxima es $1.5\ \mathrm{V}$ a $150\ ^\circ\mathrm{C}$, por lo que no daña el pin del ESP32.

## 3. Procesamiento Digital por Software

1. **Escalamiento:** Dividir los milivoltios entre `10.0`, utilizando punto flotante (`float`) para no perder decimales.

2. **Filtrado (Media Móvil):** Promediar las últimas $N$ muestras ($N=10$ a $10\ \mathrm{Hz}$ equivale al último segundo) para suavizar el ruido.

   - Se implementa con un **búfer circular**, donde el dato nuevo sobrescribe al más viejo.
   - Introduce un retardo de respuesta de:

     $$
     t_{\mathrm{retardo}} = N \times T_s
     $$

     Para $N=10$ y $T_s=0.1\ \mathrm{s}$:

     $$
     t_{\mathrm{retardo}} = 10 \times 0.1 = 1\ \mathrm{s}
     $$

3. **Decisión con Histéresis:** Evita el parpadeo errático de una alarma por ruido usando **dos umbrales** distintos. Por ejemplo, encender a $28\ ^\circ\mathrm{C}$ y apagar al bajar a $26\ ^\circ\mathrm{C}$, en lugar de utilizar un solo umbral.

## 4. Visualización en Pantalla LCD 16x2

### 4.1. Arquitectura Interna

- **Estructura:** Tiene **16 columnas** (`0` a `15`) y **2 filas** (`0` y `1`), sumando 32 casillas. Cada casilla es una matriz de $5 \times 8$ puntos (la última fila es para el cursor).

- **Controlador HD44780:** Recibe códigos **ASCII** (por ejemplo, `65` para `'A'`) desde el ESP32 y gestiona tres memorias:

  - **CGROM:** Formas de letras estándar grabadas de fábrica.
  - **DDRAM:** Códigos de los caracteres que se muestran en cada casilla.
  - **CGRAM:** Espacio para crear hasta **8 caracteres propios** (`0` a `7`), como el símbolo de grados (`°`), usando `createChar()` y mostrándolo con `write()`.

### 4.2. Conexión Directa vs. Módulo I2C

Para comunicarse con la LCD se utilizan pines de control y datos:

- **RS:** Selecciona entre comando (`0`) o dato (`1`).
- **R/W:** Se conecta a GND para trabajar únicamente en modo escritura.
- **E (Enable):** Funciona como botón de envío mediante un flanco de bajada.

| Aspecto | Conexión Directa (Modo 4 bits) | Con Módulo I2C (Chip PCF8574) |
|:---|:---|:---|
| **Pines del ESP32** | **6 pines** (RS, E, D4–D7) | **2 pines** (SDA en GPIO21, SCL en GPIO22) |
| **Cableado total** | Aproximadamente 12 cables + potenciómetro | **4 cables** (GND, VCC a 5V, SDA, SCL) |
| **Contraste** | Potenciómetro externo en pin VO | Tornillo azul integrado en el módulo |
| **Identificación** | Directa por hardware | Dirección I2C (`0x27` o `0x3F`) |

**Nota:** Aunque el módulo I2C se alimenta a $5\ \mathrm{V}$, la corriente de retorno hacia los pines de $3.3\ \mathrm{V}$ del ESP32 a través de sus resistencias *pull-up* es muy baja (aproximadamente $0.2\ \mathrm{mA}$), por lo que la conexión directa es segura.

### 4.3. Programación y Buenas Prácticas

Con la librería `LiquidCrystal_I2C` se utilizan las siguientes funciones:

- `lcd.init()`: Inicializa la pantalla.
- `lcd.setCursor(col, fila)`: Ubica el cursor en una columna y fila determinadas.
- `lcd.print()`: Muestra información en la pantalla.

**Buenas prácticas:**

- **Separar lo fijo de lo variable:** Escribir textos estáticos (`"Temp:"`, `"Alarma:"`) una vez en `setup()` y actualizar únicamente los datos cambiantes en `loop()`.

- **Evitar parpadeos y restos:** No usar `lcd.clear()` en `loop()` porque demora aproximadamente $2\ \mathrm{ms}$ y hace parpadear la pantalla. Para evitar dígitos residuales al pasar de números largos a cortos (por ejemplo, `"25.10"` a `"9.5"`, que podría dejar `"9.510"`), sobrescribir rellenando con espacios en blanco.

- **Sostenibilidad:** Apagar la luz de fondo (`lcd.noBacklight()`) cuando no se observa la pantalla y encenderla únicamente ante alarmas o pulsaciones para ahorrar energía.
