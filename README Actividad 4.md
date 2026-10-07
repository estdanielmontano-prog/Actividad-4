# Control de LEDs por gestos de la mano (MediaPipe + ESP32)

Práctica de Micros: la cámara del computador reconoce gestos de la mano con **MediaPipe Gesture Recognizer** y el programa en **Python** envía un comando por **UART (USB)** a una **ESP32**, que enciende tres LEDs con distintos niveles de intensidad (PWM) o ejecuta secuencias de luces.

---

## 1. Objetivo

- Reconocer gestos de una o dos manos en tiempo real con la cámara del PC.
- Enviar al ESP32 el comando correspondiente a cada gesto por el puerto serie.
- Controlar la intensidad de tres LEDs con PWM (30 %, 70 % y 100 %).
- Ejecutar dos secuencias de luces que actúen como "interrupciones" al cambiar de gesto.

## 2. Descripción del sistema

| Gesto (MediaPipe) | Comando serial | Acción en la ESP32 |
|---|---|---|
| `Closed_Fist` (puño) | `NIVEL_30` | LED amarillo al **30 %** |
| `Victory` (paz) | `NIVEL_70` | LED azul al **70 %** |
| 2 × `Open_Palm` (dos manos abiertas) | `NIVEL_100` | LED rojo al **100 %** |
| `Thumb_Down` (pulgar abajo) | `MODO_1` | Secuencia 1: alternancia azul / rojo |
| `Thumb_Up` (pulgar arriba) | `MODO_2` | Secuencia 2: barrido amarillo → azul → rojo |

Las secuencias se repiten hasta que se reconozca otro gesto, momento en el que se interrumpen y se aplica el nuevo comando.

## 3. Hardware

- 1 × ESP32 DevKit
- 3 × LED (amarillo, azul y rojo)
- 3 × resistencias de 220 Ω
- Protoboard y cables Dupont
- Cable USB (alimentación + comunicación serial)
- Cámara web del computador

### Conexiones

Cada LED va del pin indicado → resistencia de 220 Ω → ánodo; el cátodo va a **GND**.

| LED | Pin ESP32 | Salida |
|---|---|---|
| Amarillo | GPIO25 | PWM (5 kHz, 8 bits) |
| Azul | GPIO26 | PWM (5 kHz, 8 bits) |
| Rojo | GPIO27 | PWM (5 kHz, 8 bits) |

## 4. Arquitectura

```
Cámara PC ──► OpenCV (espejo + RGB) ──► MediaPipe Gesture Recognizer ──► Lógica de comandos
                                                                              │
                                                                  UART 115200 │ "NIVEL_30\n"
                                                                              ▼
                                                              ESP32 ──► PWM / secuencias ──► LEDs
```

**Protocolo:** una línea de texto terminada en `\n` con el nombre del comando (`NIVEL_30`, `NIVEL_70`, `NIVEL_100`, `MODO_1`, `MODO_2`). La ESP32 responde `OK <comando>` cuando lo ejecuta.

## 5. Estructura del proyecto

```
Actividad4_Gestos_LED/
├── firmware/
│   └── gestos_leds/gestos_leds.ino     # Código del ESP32
├── python/
│   ├── reconocedor_gestos.py           # Cámara + MediaPipe + serial
│   ├── gesture_recognizer.task         # Modelo de MediaPipe
│   └── requirements.txt
├── docs/                               # Enunciado y foto del montaje
├── video/                              # Video de funcionamiento
└── README.md
```

## 6. Cómo funciona el código

### 6.1 Firmware ESP32 (`gestos_leds.ino`)
1. Configura los tres pines como canales PWM (`ledcAttach`, 5 kHz, 8 bits) con el core ESP32 v3.x.
2. Lee líneas por serial y las normaliza (`trim()` y mayúsculas).
3. Los niveles fijos se convierten de porcentaje a ciclo de trabajo (`pct × 255 / 100`) y se aplican a un solo LED, apagando los demás.
4. Las secuencias usan una **máquina de estados con `millis()`**, sin `delay()`, por lo que el programa nunca se bloquea y cualquier comando nuevo interrumpe la secuencia en curso.

### 6.2 Script Python (`reconocedor_gestos.py`)
1. Captura el video con OpenCV y lo voltea para efecto espejo.
2. Pasa cada cuadro a `GestureRecognizer` (hasta 2 manos) y obtiene el nombre del gesto de cada mano.
3. `decidir_comando()` traduce los gestos a un comando: dos palmas abiertas dan `NIVEL_100`; con una mano se usa un diccionario gesto → comando.
4. **Antirrebote:** un gesto debe mantenerse 5 cuadros seguidos antes de enviarse, para evitar comandos falsos por detecciones momentáneas.
5. La clase `Enlace` escribe por serial solo cuando el comando cambia, sin saturar el puerto.
6. Muestra en pantalla los gestos detectados y el último comando enviado. Se sale con la tecla `q`.


## Referencias
- [MediaPipe Gesture Recognizer – demo web](https://google-ai-edge.github.io/mediapipe-samples-web/#/vision/gesture_recognizer)
- [Documentación de MediaPipe Gesture Recognizer](https://ai.google.dev/edge/mediapipe/solutions/vision/gesture_recognizer)
