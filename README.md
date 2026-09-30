# tarea-4-micros
# 💡 Control de Iluminación por Gestos de la Mano

Sistema de control de iluminación basado en el reconocimiento de gestos de la mano. Una página web detecta la mano con **MediaPipe Hands** a través de la cámara, clasifica el gesto y envía el comando por **WiFi (HTTP)** a un **ESP32**, que controla tres LEDs mediante **PWM**.

> Proyecto desarrollado para la asignatura de Microcontroladores — **Universidad Militar Nueva Granada**.

---

## 🎥 Video de verificación

[![Video demo del sistema](https://img.youtube.com/vi/tAfyYx-zgt4/0.jpg)](https://youtube.com/shorts/tAfyYx-zgt4)

▶️ **Ver el video en YouTube:** https://youtube.com/shorts/tAfyYx-zgt4

---

## 📋 Descripción del problema

Con la librería MediaPipe se debe desarrollar un sistema de control de iluminación mediante gestos de la mano, usando un ESP32 y tres LEDs:

| Gesto | Acción |
|-------|--------|
| ✊ Puño cerrado | LED **amarillo** al **30 %** de intensidad |
| ✌️ Señal de paz | LED **azul** al **70 %** de intensidad |
| 🖐️ Mano abierta | LED **rojo** al **100 %** de intensidad |
| 👎 Pulgar abajo | **Primera interrupción:** secuencia de luces (**Modo 1**) |
| 👍 Pulgar arriba | **Segunda interrupción:** secuencia de luces (**Modo 2**) |
| 🚫 Sin mano (> 1 s) | Todos los LEDs se apagan |

---

## 🏗️ Arquitectura

```
┌──────────┐    ┌───────────────────────┐    HTTP GET     ┌──────────┐    PWM    ┌──────┐
│  Cámara  │──▶ │  Navegador (index.html)│ ─────────────▶ │  ESP32   │ ────────▶ │ LEDs │
│  (webcam)│    │  MediaPipe Hands + JS │  /gesture?...   │ WebServer│           │ Y B R│
└──────────┘    └───────────────────────┘                 └──────────┘           └──────┘
```

1. La cámara captura el video en el navegador.
2. **MediaPipe Hands** obtiene los 21 puntos (landmarks) de la mano.
3. Un clasificador propio en JavaScript determina el gesto a partir de esos puntos.
4. El gesto se envía al ESP32 con una petición `HTTP GET`.
5. El ESP32 enciende el LED correspondiente con PWM o ejecuta la secuencia del modo.

---

## 🧰 Materiales

- 1 × ESP32 (DevKit)
- 3 × LED (amarillo, azul, rojo)
- 3 × resistencias de 220 Ω – 330 Ω
- Protoboard y cables
- Computador con cámara web y navegador (Chrome/Edge recomendado)
- Red WiFi compartida entre el computador y el ESP32

### 🔌 Conexiones

| LED | Pin ESP32 | Canal PWM |
|-----|-----------|-----------|
| 🟡 Amarillo | GPIO 26 | 0 |
| 🔵 Azul | GPIO 25 | 1 |
| 🔴 Rojo | GPIO 33 | 2 |

Cada LED va conectado: `GPIO → resistencia → ánodo del LED → cátodo → GND`.

---

## 📁 Estructura del repositorio

```
├── index.html            # Interfaz web + detección y clasificación de gestos
├── esp32_gestos.ino      # Firmware del ESP32 (servidor web + control de LEDs)
└── README.md
```

---

## 🧠 Desarrollo

### 1. Detección de gestos (`index.html`)

Se usa `@mediapipe/hands` con un máximo de 1 mano, `modelComplexity: 1` y confianza mínima de 0.7.

La clasificación se hace con los landmarks (0 = muñeca, 4/8/12/16/20 = puntas de los dedos):

- **Dedo doblado:** la distancia punta–muñeca es menor a `0.65 ×` el tamaño de la mano (distancia muñeca–`INDEX_MCP`), **o** la punta está por debajo de la articulación PIP.
- **Pulgar:** se evalúa aparte por distancia a la muñeca y por su dirección vertical (arriba/abajo).

Reglas:

| Gesto | Condición |
|-------|-----------|
| `fist` | Los 4 dedos y el pulgar doblados |
| `peace` | Índice y medio extendidos; anular y meñique doblados |
| `open_hand` | Los 4 dedos extendidos |
| `thumbs_up` | Pulgar extendido hacia arriba + 4 dedos doblados |
| `thumbs_down` | Pulgar extendido hacia abajo + 4 dedos doblados |

**Estabilización:** un gesto solo se acepta tras **5 fotogramas consecutivos** iguales (`STABLE_FRAMES = 5`), evitando falsos positivos.

**Apagado automático:** si se pierde la mano durante **1 segundo** (`NO_HAND_DELAY`), se envía `gesture=off` y se apagan los LEDs.

**Modos 1 y 2:** el comando se envía **una sola vez** al activarse (`modeActivated`), y la secuencia corre de forma autónoma en el ESP32.

### 2. Firmware del ESP32 (`esp32_gestos.ino`)

- Se conecta a WiFi y levanta un `WebServer` en el puerto 80.
- Usa 3 canales PWM (5 kHz, 8 bits) con `ledcSetup` / `ledcAttachPin`.
- **Modo normal:** enciende un único LED con la intensidad del gesto (`map(porcentaje, 0, 100, 0, 255)`).
- **Modo 1** (paso cada 500 ms): 🟡30 % → 🔵70 % → 🔴100 % → 🔵70 % → 🟡30 %
- **Modo 2** (paso cada 400 ms): 🔴100 % → 🔵70 % → 🟡30 % → todos ON → todos OFF

### 3. API HTTP del ESP32

| Endpoint | Descripción |
|----------|-------------|
| `GET /gesture?gesture=<g>&intensity=<0-100>` | Aplica el gesto (`fist`, `peace`, `open_hand`, `thumbs_down`, `thumbs_up`) |
| `GET /status` | Devuelve el gesto, intensidad y modo actuales (JSON) |
| `GET /off` | Apaga todos los LEDs |

Ejemplo:

```
http://192.168.40.87/gesture?gesture=peace&intensity=70
→ {"status":"ok","gesture":"peace","intensity":70,"mode":"normal"}
```

---

## 🚀 Instalación y uso

### ESP32

1. Instala el **Arduino IDE** y el soporte de placas ESP32 (paquete *esp32 by Espressif*, versión **2.x**, ya que se usa la API `ledcSetup`/`ledcAttachPin`).
2. Abre `esp32_gestos.ino` y cambia tus credenciales:
   ```cpp
   const char* ssid = "TU_RED_WIFI";
   const char* password = "TU_CONTRASEÑA";
   ```
3. Selecciona la placa (*ESP32 Dev Module*) y sube el código.
4. Abre el Monitor Serial (115200) y anota la **IP** que aparece.

### Página web

1. En `index.html`, actualiza la IP del ESP32:
   ```js
   const ESP32_IP = 'http://TU_IP_DEL_ESP32';
   ```
2. Sirve la página desde `localhost` (la cámara requiere `localhost` o HTTPS, y el ESP32 responde por HTTP):
   ```bash
   python -m http.server 8000
   ```
3. Abre `http://localhost:8000`, pulsa **▶️ Iniciar Cámara** y acepta los permisos.
4. Haz los gestos frente a la cámara. 🎉

> ⚠️ El computador y el ESP32 deben estar en la **misma red WiFi**.

---

## 🛠️ Tecnologías

- [MediaPipe Hands](https://google-ai-edge.github.io/mediapipe-samples-web/#/vision/gesture_recognizer) — detección de landmarks de la mano
- HTML5 / CSS3 / JavaScript
- ESP32 + Arduino framework (`WiFi.h`, `WebServer.h`)
- PWM con LEDC

---

## 👤 Autor

**Carlos** — Universidad Militar Nueva Granada
Asignatura: Microcontroladores
