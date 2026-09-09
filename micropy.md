# Python-First Embedded & Robotics Architecture

> **Leitphilosophie:** Verhalten und Intelligenz in Python – Hardware-Abstraktion in C.  
> MicroPython überbrückt beide Welten direkt auf dem Mikrocontroller.

---

## Grundsatz

Dieses Projekt folgt dem Prinzip, Komplexität dort zu halten, wo sie beherrschbar ist:

- **C** für das absolut Notwendige: zeitkritische Interrupts, Treiber, HAL
- **MicroPython / Python** für alles andere: Logik, Verhalten, Sensorverarbeitung, KI, Kommunikation
- **Kein C++** – die zusätzliche Sprachkomplexität (Templates, vtables, Ausnahmebehandlung) rechtfertigt sich im Embedded-Kontext selten

---

## Architekturübersicht

```
┌─────────────────────────────────────────────────────┐
│              Anwendungsebene (Python)                │
│   Verhalten · Zustandsmaschinen · KI-Inferenz       │
│   Missionsplanung · Kommunikationslogik             │
├─────────────────────────────────────────────────────┤
│           MicroPython-Laufzeitumgebung               │
│     MicroPython-VM · asyncio · ulab / numpy         │
├─────────────────────────────────────────────────────┤
│         Hardware-Abstraktionsschicht (C)             │
│   Treiber · ISR · RTOS-Hooks · Peripherie-HAL       │
├─────────────────────────────────────────────────────┤
│                    Hardware                          │
│    MCU · Sensoren · Aktoren · Kommunikation          │
└─────────────────────────────────────────────────────┘
```

---

## Schichtbeschreibung

### Anwendungsebene – Python / MicroPython

Hier lebt die gesamte Intelligenz und das Verhalten des Systems.

**Zuständigkeiten:**
- Zustandsmaschinen und Verhaltenssteuerung
- Sensorverarbeitung und Datenfusion
- KI-Inferenz (TensorFlow Lite Micro via `ulab`, ONNX-Modelle)
- Missionsplanung und Entscheidungslogik
- Kommunikationsprotokolle (MQTT, WebSocket, ROS2-rclpy)
- Schnelle Iteration und Prototypentwicklung

**Typische Bibliotheken:**
```python
import asyncio          # Asynchrone Aufgabensteuerung
import ulab.numpy as np # Vektorrechnung auf MCU
import machine          # Hardware-Abstraktion (MicroPython)
import ujson            # JSON-Verarbeitung
import umqtt.simple     # MQTT-Kommunikation
```

**Beispiel – asynchrone Sensorschleife:**
```python
import asyncio
from machine import I2C, Pin

async def sensor_loop(sensor, controller):
    while True:
        data = sensor.read()
        await controller.process(data)
        await asyncio.sleep_ms(10)

async def main():
    sensor = IMUSensor(I2C(0, scl=Pin(22), sda=Pin(21)))
    controller = BehaviorController()
    await asyncio.gather(
        sensor_loop(sensor, controller),
        controller.run()
    )

asyncio.run(main())
```

---

### Hardware-Abstraktionsschicht – C

Nur dort C einsetzen, wo Python an physikalische Grenzen stößt.

**Zuständigkeiten:**
- Zeitkritische Interrupt-Service-Routinen (ISR)
- Low-Level-Gerätetreiber (SPI, I2C, UART, PWM)
- RTOS-Integration (FreeRTOS, Zephyr)
- MicroPython C-Erweiterungsmodule
- Echtzeit-Regelkreise (PID mit μs-Präzision)

**Kriterien für C-Code:**
```
Zeitanforderung < 1ms      → C
Direkter Registerzugriff   → C
ISR / Hardware-Interrupt   → C
Alles andere               → Python / MicroPython
```

**Beispiel – C-Erweiterungsmodul für MicroPython:**
```c
/* encoder_module.c – wird als MicroPython-Modul eingebunden */
#include "py/runtime.h"
#include "py/obj.h"

static volatile int32_t encoder_count = 0;

void IRAM_ATTR encoder_isr(void *arg) {
    /* Zeitkritisch: nur Zählen, nichts weiter */
    encoder_count += gpio_get_level(ENC_B) ? 1 : -1;
}

STATIC mp_obj_t get_count(void) {
    return mp_obj_new_int(encoder_count);
}
STATIC MP_DEFINE_CONST_FUN_OBJ_0(get_count_obj, get_count);

/* Modul-Registrierung → import encoder in MicroPython */
STATIC const mp_rom_map_elem_t encoder_module_globals_table[] = {
    { MP_ROM_QSTR(MP_QSTR___name__), MP_ROM_QSTR(MP_QSTR_encoder) },
    { MP_ROM_QSTR(MP_QSTR_get_count), MP_ROM_PTR(&get_count_obj) },
};
```

---

## Zielplattformen

| Plattform | Laufzeit | Einsatz |
|---|---|---|
| **ESP32 / ESP32-S3** | MicroPython | Hauptcontroller, WLAN/BT |
| **RP2040 (Raspberry Pi Pico)** | MicroPython | Motorsteuerung, Sensorhubs |
| **OpenMV Cam** | MicroPython | Computer Vision direkt auf MCU |
| **STM32** | MicroPython + C-HAL | Industrielle Anwendungen |
| **Raspberry Pi** | CPython | Hochlevelverarbeitung, ROS2 |
| **Linux-SBC (Jetson, etc.)** | CPython | KI-Inferenz, Missionsplanung |

---

## Warum nicht C++?

| Argument | Realität im Embedded-Kontext |
|---|---|
| „Performance" | I/O-Wartezeiten dominieren – Sprachoverhead irrelevant |
| „Memory Management" | C++ Pointer-Bugs sind die häufigste Fehlerquelle |
| „Skalierbarkeit" | Python-Module skalieren besser als C++-Templates |
| „Industrie-Standard" | ROS2 unterstützt Python vollständig (rclpy) |
| „Hardware-Nähe" | C reicht vollständig – C++ fügt unnötige Komplexität hinzu |

**C++ Probleme die entfallen:**
- Keine vtable-Overhead-Diskussionen
- Kein undefined behavior durch falsche Pointer-Arithmetik
- Keine Template-Fehler zur Compilezeit
- Kein manuelles Speichermanagement im Anwendungscode

---

## Entwicklungsworkflow

```
1. Prototyp in MicroPython (REPL / Thonny)
        ↓
2. Validierung der Logik in Python (PC, pytest)
        ↓
3. Deployment auf Zielplattform (mpremote / rshell)
        ↓
4. Profiling: Wo sind echte Engpässe?
        ↓
5. Nur bei Bedarf: C-Erweiterung für kritische Pfade
        ↓
6. Wiederverwendung: C-Modul bleibt minimal,
   Python-Schicht wächst
```

---

## Toolchain

```bash
# MicroPython-Firmware flashen (ESP32)
esptool.py --chip esp32 erase_flash
esptool.py --chip esp32 write_flash -z 0x1000 micropython.bin

# Dateien übertragen
mpremote connect /dev/ttyUSB0 cp main.py :main.py
mpremote connect /dev/ttyUSB0 run main.py

# REPL für schnelles Testen
mpremote connect /dev/ttyUSB0 repl

# Unit-Tests auf PC (gleicher Python-Code)
pytest tests/ -v

# C-Erweiterungen bauen (ESP-IDF)
idf.py build && idf.py flash
```

---

## Projektstruktur

```
project/
├── src/
│   ├── main.py              # Einstiegspunkt (MicroPython)
│   ├── behavior/            # Verhaltenslogik (Python)
│   │   ├── controller.py
│   │   ├── state_machine.py
│   │   └── planner.py
│   ├── sensors/             # Sensorabstraktion (Python)
│   │   ├── imu.py
│   │   ├── lidar.py
│   │   └── camera.py
│   ├── actuators/           # Aktorsteuerung (Python)
│   │   ├── motor.py
│   │   └── servo.py
│   └── hal/                 # Hardware-Abstraktion (C)
│       ├── encoder.c
│       ├── encoder.h
│       └── pwm_driver.c
├── tests/                   # PC-seitige Tests (pytest)
│   ├── test_behavior.py
│   └── test_sensors.py
├── tools/
│   └── deploy.sh            # mpremote Deploy-Skript
├── firmware/                # Vorkompilierte MicroPython-Binaries
└── README.md
```

---

## Philosophie in einem Satz

> **Schreibe so viel Python wie möglich, so viel C wie nötig – und niemals C++.**

---

## Weiterführende Ressourcen

- [MicroPython Dokumentation](https://docs.micropython.org)
- [ulab – NumPy für MicroPython](https://github.com/v923z/micropython-ulab)
- [mpremote – Offizielles Deploy-Tool](https://docs.micropython.org/en/latest/reference/mpremote.html)
- [OpenMV – MicroPython Computer Vision](https://openmv.io)
- [ROS2 rclpy – Python-API für ROS2](https://docs.ros2.org/latest/api/rclpy/)
- [ESP-IDF MicroPython C-Erweiterungen](https://docs.micropython.org/en/latest/develop/cmodules.html)
