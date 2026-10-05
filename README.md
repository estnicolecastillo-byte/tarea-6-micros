# Actividad: ESP32, comunicación I²C / SPI / UART y visión computacional
## NICOLE NATALIA CASTILLO 
## KEVIN ALEJANDRO VEGA MEDINA


## Punto 1: Teclado I²C + brazo robótico en PyBullet

### Descripción
Un ESP32 lee un teclado matricial 4x4 por I²C y muestra la tecla en una pantalla LCD. La tecla se envía por UART al PC, donde un script en Python mueve el brazo robótico (`brazo.urdf` de U_Militar) en PyBullet para **dibujar el número o carácter presionado**.

```
Teclado 4x4 (I²C) → ESP32 → LCD I²C
                       └──→ UART → Python → PyBullet (brazo dibujando)
```

### Hardware
- ESP32
- Teclado matricial 4x4 (módulo I²C) 
- LCD 16x2 con módulo I²C
- Protoboard y cables

### Conexiones ⚠️ (ajusta a tu montaje)
| Componente | Pin ESP32 |
|------------|-----------|
| SDA (teclado y LCD) | GPIO 21 |
| SCL (teclado y LCD) | GPIO 22 |
| VCC | 3V3 / 5V según el módulo |
| GND | GND |

Direcciones I²C típicas: LCD `0x27`, teclado con PCF8574 `0x20`.

<img width="831" height="513" alt="image" src="https://github.com/user-attachments/assets/5e3b553f-9a45-450e-891e-aaee0db64c4b" />


### Funcionamiento
El ESP32 consulta el teclado 4x4 por el bus I²C (SDA/SCL). Con un módulo PCF8574 el ESP32 le pregunta qué fila y columna están activas y obtiene la tecla (1, 5, A, #...).
Cada tecla se muestra en la LCD 16x2, que comparte el mismo bus I²C con otra dirección.
El ESP32 manda la tecla por UART (USB) a 115200 baudios, por ejemplo como un carácter y un salto de línea.
 Un script de Python con pyserial lee ese carácter. Busca la trayectoria del número, que es una lista de puntos (x, y) sobre el plano de dibujo. Con PyBullet calcula la cinemática inversa para que la punta del brazo (brazo.urdf) pase por esos puntos, y va moviendo las articulaciones.
Resultado. El brazo traza el número en la simulación, como en la imagen del enunciado.


![Simulación Punto 1](docs/simulacion_punto1.png)

---

## Punto 2: Reconocimiento de dígitos con OpenCV + SPI + OLED I²C

### Descripción
El PC captura con la cámara un dígito escrito a mano, lo preprocesa con OpenCV, lo clasifica con una CNN y envía el resultado por puerto serie al **ESP-A (maestro SPI)**. Este lo reenvía por SPI al **ESP-B (esclavo SPI)**, que lo muestra en una **pantalla OLED I²C**.

```
Cámara PC → Preprocesamiento OpenCV → CNN → Puerto serie → ESP-A (maestro SPI) → SPI → ESP-B (esclavo SPI) → OLED I²C
```

### Protocolos
| Enlace | Formato |
|--------|---------|
| PC → ESP-A (USB serie, 115200) | Carácter ASCII `'0'`–`'9'` + `\n` |
| ESP-A → ESP-B (SPI modo 0, 1 MHz) | 4 bytes: `[0xA5, dígito, 0xA5 ^ dígito, 0x00]` |
| ESP-B → OLED | I²C, SSD1306 128x64, dirección `0x3C` |

### Conexiones

**SPI (ESP-A maestro → ESP-B esclavo)**

| Señal | ESP-A (ESP32) | ESP-B (ESP32) | ESP-B (ESP32-C3) |
|-------|---------------|---------------|------------------|
| SCK   | GPIO 18 | GPIO 18 | GPIO 4 |
| MOSI  | GPIO 23 | GPIO 23 | GPIO 6 |
| MISO  | GPIO 19 | GPIO 19 | GPIO 5 |
| CS    | GPIO 5  | GPIO 5  | GPIO 7 |
| GND   | GND     | GND     | GND    |

**OLED en ESP-B**

| OLED | ESP32 | ESP32-C3 |
|------|-------|----------|
| SDA  | GPIO 21 | GPIO 8 |
| SCL  | GPIO 22 | GPIO 9 |
| VCC  | 3V3 | 3V3 |
| GND  | GND | GND |

Los pines se pueden cambiar en los `#define` al inicio de cada `.ino`.

<img width="764" height="400" alt="image" src="https://github.com/user-attachments/assets/ccb4b64f-14c1-4990-9199-3b0feb24059d" />


### codigo maestro
```cpp
/*
 * ESP-A  (MAESTRO SPI)
 * Recibe por puerto serie (USB, desde el PC) un dígito ASCII '0'..'9'
 * y lo envía por SPI al ESP-B con una trama de 4 bytes:
 *   [0xA5, dígito, 0xA5 ^ dígito, 0x00]
 */
#include <SPI.h>

#define PIN_SCK   18
#define PIN_MISO  19
#define PIN_MOSI  23
#define PIN_CS     5

#define SPI_HZ    1000000
#define CABECERA  0xA5

void enviarSPI(uint8_t digito) {
  uint8_t trama[4] = { CABECERA, digito, (uint8_t)(CABECERA ^ digito), 0x00 };

  SPI.beginTransaction(SPISettings(SPI_HZ, MSBFIRST, SPI_MODE0));
  digitalWrite(PIN_CS, LOW);
  delayMicroseconds(50);
  SPI.transferBytes(trama, nullptr, 4);
  delayMicroseconds(50);
  digitalWrite(PIN_CS, HIGH);
  SPI.endTransaction();
}

void setup() {
  Serial.begin(115200);
  pinMode(PIN_CS, OUTPUT);
  digitalWrite(PIN_CS, HIGH);
  SPI.begin(PIN_SCK, PIN_MISO, PIN_MOSI, PIN_CS);
  Serial.println("ESP-A listo (maestro SPI)");
}

void loop() {
  while (Serial.available()) {
    char c = Serial.read();
    if (c >= '0' && c <= '9') {
      uint8_t d = c - '0';
      enviarSPI(d);
      Serial.printf("SPI -> %u\n", d);
    }
    // '\n', '\r' y cualquier otro carácter se ignoran
  }
}
```
### Uso
OpenCV lee la cámara y toma un recuadro central de la imagen.
 Convierte a gris, desenfoca un poco, aplica umbral adaptativo y limpia el ruido. Queda el trazo en blanco sobre fondo negro, que es el formato de MNIST, la base de datos con la que se entrenó la red.
 Recorta el dígito, lo escala a 20x20 y lo centra en un lienzo de 28x28. La red convolucional devuelve una probabilidad para cada dígito del 0 al 9. Se toma la mayor.
Filtro de estabilidad. Solo acepta la predicción si la confianza es mayor al 85 % y se repite en 8 frames seguidos. Así no se envían lecturas falsas.
Envío. Manda el dígito por puerto serie al ESP-A, una sola vez por cada cambio.
![Reconocimiento Punto 2](docs/reconocimiento_punto2.png)

### codigo esclavo
```cpp
/*
 * ESP-B  (ESCLAVO SPI + pantalla OLED I2C SSD1306 128x64)
 * Recibe del ESP-A la trama [0xA5, dígito, 0xA5 ^ dígito, 0x00]
 * y muestra el dígito en grande en la OLED.
 *
 * Librerías (Gestor de librerías de Arduino IDE):
 *   - Adafruit SSD1306
 *   - Adafruit GFX Library
 */
#include <Arduino.h>
#include <Wire.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>
#include "driver/spi_slave.h"

// ---------- Pines ----------
#if CONFIG_IDF_TARGET_ESP32C3
  // ESP32-C3 (como en el diagrama)
  #define HOST_SPI  SPI2_HOST
  #define PIN_SCK   4
  #define PIN_MISO  5
  #define PIN_MOSI  6
  #define PIN_CS    7
  #define PIN_SDA   8
  #define PIN_SCL   9
#else
  // ESP32 clásico
  #define HOST_SPI  SPI3_HOST
  #define PIN_SCK   18
  #define PIN_MISO  19
  #define PIN_MOSI  23
  #define PIN_CS     5
  #define PIN_SDA   21
  #define PIN_SCL   22
#endif

#define CABECERA     0xA5
#define OLED_ANCHO   128
#define OLED_ALTO    64
#define OLED_ADDR    0x3C

Adafruit_SSD1306 oled(OLED_ANCHO, OLED_ALTO, &Wire, -1);

WORD_ALIGNED_ATTR uint8_t rxbuf[4];
WORD_ALIGNED_ATTR uint8_t txbuf[4];

void mostrarTexto(const char *msg) {
  oled.clearDisplay();
  oled.setTextSize(1);
  oled.setTextColor(SSD1306_WHITE);
  oled.setCursor(0, 0);
  oled.print(msg);
  oled.display();
}

void mostrarDigito(uint8_t d) {
  oled.clearDisplay();
  oled.setTextSize(6);                 // caracter de 36x48 px
  oled.setTextColor(SSD1306_WHITE);
  oled.setCursor((OLED_ANCHO - 36) / 2, (OLED_ALTO - 48) / 2);
  oled.print(d);
  oled.display();
}

void setup() {
  Serial.begin(115200);

  Wire.begin(PIN_SDA, PIN_SCL);
  if (!oled.begin(SSD1306_SWITCHCAPVCC, OLED_ADDR)) {
    Serial.println("No se encontró la OLED (revisa SDA/SCL y dirección 0x3C)");
    while (true) delay(1000);
  }
  mostrarTexto("ESP-B listo\nEsperando digito...");

  spi_bus_config_t bus = {};
  bus.mosi_io_num = PIN_MOSI;
  bus.miso_io_num = PIN_MISO;
  bus.sclk_io_num = PIN_SCK;
  bus.quadwp_io_num = -1;
  bus.quadhd_io_num = -1;

  spi_slave_interface_config_t esclavo = {};
  esclavo.mode = 0;
  esclavo.spics_io_num = PIN_CS;
  esclavo.queue_size = 3;
  esclavo.flags = 0;

  esp_err_t r = spi_slave_initialize(HOST_SPI, &bus, &esclavo, SPI_DMA_CH_AUTO);
  if (r != ESP_OK) {
    Serial.printf("Error al iniciar SPI esclavo: %d\n", r);
    while (true) delay(1000);
  }
  Serial.println("ESP-B listo (esclavo SPI)");
}

void loop() {
  memset(rxbuf, 0, sizeof(rxbuf));
  memset(txbuf, 0, sizeof(txbuf));

  spi_slave_transaction_t t = {};
  t.length    = 4 * 8;        // bits
  t.rx_buffer = rxbuf;
  t.tx_buffer = txbuf;

  // Bloquea hasta que el maestro complete una transacción
  esp_err_t r = spi_slave_transmit(HOST_SPI, &t, portMAX_DELAY);
  if (r != ESP_OK) return;

  bool valida = (rxbuf[0] == CABECERA) &&
                (rxbuf[1] <= 9) &&
                (rxbuf[2] == (uint8_t)(CABECERA ^ rxbuf[1]));
  if (valida) {
    Serial.printf("Recibido por SPI: %u\n", rxbuf[1]);
    mostrarDigito(rxbuf[1]);
  } else {
    Serial.println("Trama SPI invalida");
  }
}
```




## Autor
NICOLE NATALIA CASTILLO
KEVIN ALEJANDRO VEGA MEDINA
