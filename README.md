# Actividad: ESP32, comunicación I²C / SPI / UART y visión computacional

## Estructura del repositorio


## Instalación general


## Punto 1: Teclado I²C + brazo robótico en PyBullet

### Descripción
Un ESP32 lee un teclado matricial 4x4 por I²C y muestra la tecla en una pantalla LCD. La tecla se envía por UART al PC, donde un script en Python mueve el brazo robótico (`brazo.urdf` de U_Militar) en PyBullet para **dibujar el número o carácter presionado**.

```
Teclado 4x4 (I²C) → ESP32 → LCD I²C
                       └──→ UART → Python → PyBullet (brazo dibujando)
```

### Hardware
- ESP32
- Teclado matricial 4x4 (módulo I²C) ⚠️
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

![Esquemático Punto 1](docs/esquematico_punto1.png)

### Uso
1. Carga `punto1_teclado_brazo/firmware/esp32_teclado/esp32_teclado.ino` en el ESP32.
2. Edita el puerto serie (ej. `COM7`) y los baudios (`115200`) en `simulacion/dibujar_brazo.py`.
3. Cierra el Monitor Serie del Arduino IDE y ejecuta:
```bash
   cd punto1_teclado_brazo/simulacion
   python dibujar_brazo.py
```
4. Presiona una tecla del teclado: aparece en la LCD y el brazo dibuja el número en PyBullet.

### Funcionamiento
Describe aquí con tus palabras: cómo se lee el teclado, qué se muestra en la LCD, qué formato tiene el mensaje UART y cómo el script convierte cada tecla en una trayectoria del brazo (cinemática, articulaciones y pinza usadas).

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

![Esquemático Punto 2](docs/esquematico_punto2.png)

### Procesamiento de imagen y modelo
1. Se toma un recuadro central de la cámara.
2. Escala de grises → desenfoque gaussiano → umbral adaptativo invertido → apertura morfológica (trazo blanco sobre fondo negro, como MNIST).
3. Se recorta el dígito, se escala a 20x20 y se centra en un lienzo de 28x28.
4. La CNN (2 capas convolucionales + capa densa, entrenada con MNIST y aumento de datos) devuelve el dígito y su confianza.
5. Solo se envía cuando la predicción es estable durante 8 frames con confianza mínima del 85 %.

### Uso
1. Entrenar el modelo (una sola vez):
```bash
   cd punto2_digitos_spi/python
   python entrenar_modelo.py
```
2. Cargar `firmware/esp_b_esclavo/esp_b_esclavo.ino` en el ESP-B y `firmware/esp_a_maestro/esp_a_maestro.ino` en el ESP-A.
3. Conectar el ESP-A al PC por USB y revisar su puerto (ej. `COM7`).
4. Ejecutar el reconocimiento:
```bash
   python reconocer_digitos.py --puerto COM7
```
5. Escribe un dígito con trazo oscuro en papel blanco y ponlo dentro del recuadro. Cuando la predicción es estable, se envía y aparece en la OLED. Presiona `q` para salir.

Sin `--puerto`, el programa solo reconoce y muestra el resultado en pantalla.

![Reconocimiento Punto 2](docs/reconocimiento_punto2.png)

### Solución de problemas
- **La OLED no enciende:** revisa SDA/SCL, la dirección (`0x3C` o `0x3D`) y la alimentación.
- **No llega nada por SPI:** revisa GND común, que MOSI vaya a MOSI (sin cruzar) y que los pines coincidan con los `#define`.
- **Reconoce mal:** mejora la luz, usa trazo grueso y oscuro, y mantén el dígito centrado en el recuadro.
- **El puerto no abre:** cierra el Monitor Serie del Arduino IDE antes de ejecutar Python.

---

## Evidencias
- Punto 1: [Video de funcionamiento](docs/demo_punto1.mp4) ⚠️
- Punto 2: [Video de funcionamiento](docs/demo_punto2.mp4) ⚠️

Si un video pesa más de 25 MB, súbelo a YouTube y reemplaza el enlace.

## Autor
Natti, Ingeniería Mecatrónica ⚠️ (agrega tu nombre completo, curso y fecha)
