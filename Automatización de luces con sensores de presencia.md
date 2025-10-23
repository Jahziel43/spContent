**Nombre:** Jahziel Amado López Angulo  
**Número de Control:** 22211593  
**Correo electrónico:** l22211593@tectijuana.edu.mx  
**GitHub:** [Jahziel43](https://github.com/Jahziel43)  

---

# 💡 Proyecto: Automatización de luces con sensores de presencia  

## 🧠 Descripción general del proyecto

Este proyecto tiene como propósito la **automatización del encendido de luces mediante la detección de sonido**, utilizando un **sensor integrado en el micro:bit** que permite identificar niveles sonoros característicos de la presencia humana (como pasos, voces o apertura de puertas).

El flujo de datos recorre un **ecosistema IoT completo**, donde el micro:bit obtiene la información y esta viaja hasta un **panel en Grafana**, pasando por varias capas de procesamiento intermedio:

**Flujo general del sistema:**  
Micro:bit → MQTT (Mosquitto) → Telegraf → InfluxDB → Grafana → (Futuro: control físico de luces)

```mermaid
graph LR
  A[Micro:bit con sensor de sonido] --> B[MQTT Broker]
  B --> C[Telegraf]
  C --> D[InfluxDB]
  D --> E[Grafana]
  E --> F[Luz encendida si se detecta sonido]
 ```

> En fases posteriores, se implementará la activación automática de luces cuando el nivel de sonido supere cierto umbral configurado, lo que simulará la detección de presencia humana.

---

## ⚙️ Funcionamiento general de cada componente

### 🔹 1. Micro:bit — Captura de datos

El **micro:bit** mide el nivel de sonido ambiente mediante su sensor interno.  
Cada lectura genera un mensaje en formato **JSON**, que incluye:

- `deviceId`: identificador del dispositivo.  
- `seq`: número de secuencia de la lectura.  
- `ts_ms`: marca de tiempo en milisegundos.  
- `sound_lvl`: nivel de sonido detectado (0–255).  
- `status`: indica si la lectura está dentro del rango esperado.

El código envía estos datos por el **puerto serial USB** al computador conectado.

```javascript
function sane(value: number, lo: number, hi: number) {
    return value >= lo && value <= hi
}

let seq = 0
let payload = ""
let ts = 0
let sound = 0
let DEVICE_ID = "mbit-lab-01"

serial.redirectToUSB()
serial.setBaudRate(BaudRate.BaudRate115200)

basic.forever(function () {
    let status: string[] = []

    sound = input.soundLevel()
    ts = control.millis()

    if (!(sane(sound, 0, 255))) {
        status.push("sound_oob")
    }

    if (status.length == 0) {
        status.push("ok")
    }

    payload = "{"
    payload += "\"deviceId\":\"" + DEVICE_ID + "\","
    payload += "\"seq\":" + seq + ","
    payload += "\"ts_ms\":" + ts + ","
    payload += "\"sound_lvl\":" + sound + ","
    payload += "\"status\":[\"" + status[0] + "\"]"
    payload += "}"

    serial.writeLine(payload)

    if (sound > 120) {
        basic.showIcon(IconNames.Surprised)
    } else if (sound > 60) {
        basic.showIcon(IconNames.Happy)
    } else {
        basic.showIcon(IconNames.Asleep)
    }

    seq += 1
    basic.pause(3000)
})
```

---

### 🔹 2. Lector Python — Envío de datos por MQTT

Un script Python escucha el puerto serial donde está conectado el micro:bit.  
Cuando recibe un JSON válido, lo envía como publicación MQTT hacia el **broker Mosquitto** alojado en una **instancia EC2 de AWS**.

El script utiliza:
- `paho-mqtt` para la comunicación MQTT.
- `json` para procesar los datos.
- `TLS` y credenciales para conexión segura.

```python
import serial
import json
import time
import paho.mqtt.client as mqtt

PORT = "COM8"
BAUD = 115200

MQTT_BROKER = "100.99.60.115"
MQTT_PORT = 8883
MQTT_TOPIC = "iot/sensores"
MQTT_USER = "Jahziel"
MQTT_PASSWORD = "mosquito123"
MQTT_TLS = r"C:\Users\User\Documents\Escuela\Tecnológico\Semestre 7\Sistemas Programables\CertMQTT\mqtt.crt"

client = mqtt.Client()
client.username_pw_set(MQTT_USER, MQTT_PASSWORD)
client.tls_set(MQTT_TLS)
client.connect(MQTT_BROKER, MQTT_PORT)
client.loop_start()

print("Conectado al broker MQTT del EC2. Esperando datos del micro:bit...")

with serial.Serial(PORT, BAUD, timeout=2) as s:
    while True:
        line = s.readline().decode(errors='ignore').strip()
        if line.startswith("{") and line.endswith("}"):
            try:
                data = json.loads(line)
                sound = data.get("sound_lvl")

                payload = json.dumps({
                    "sonido": sound
                })

                client.publish(MQTT_TOPIC, payload)
                print(f"📡 Enviado MQTT: {payload}")

            except Exception as e:
                print("⚠️ Error al procesar JSON:", e)

        time.sleep(1)
```

---

### 🔹 3. Mosquitto — Servidor MQTT

Mosquitto actúa como **broker**, recibiendo mensajes del Python y reenviándolos a los suscriptores (en este caso, **Telegraf**).

Para iniciar Mosquitto en el servidor EC2:

```bash
sudo systemctl start mosquitto
sudo systemctl enable mosquitto
sudo systemctl status mosquitto
```

**Comando para visualizar los mensajes recibidos en el topic:**

```bash
mosquitto_sub -h localhost -p 8883 -t "iot/sensores" \
-u "Jahziel" -P "mosquito123" \
--cafile /etc/mosquitto/certs/mqtt.crt
```

---

### 🔹 4. Telegraf — Conector MQTT → InfluxDB

Telegraf se encarga de **leer los mensajes MQTT** y guardarlos como mediciones en **InfluxDB**.  
La configuración principal está en `/etc/telegraf/telegraf.d/mqtt.conf`.


```bash
# Input MQTT
[[inputs.mqtt_consumer]]
  servers = ["ssl://localhost:8883"]
  topics = ["iot/sensores"]
  qos = 1
  client_id = "telegraf-mqtt"
  username = "Jahziel"
  password = "mosquito123"

  data_format = "json"
  name_override = "ambiente"
  json_string_fields = []

  tls_ca = "/etc/mosquitto/certs/mqtt.crt"
  insecure_skip_verify = false

# Output InfluxDB v2
[[outputs.influxdb_v2]]
  urls = ["http://localhost:8086"]
  token = "N1hFAxCXSRCvPAA5aBbz5otNyxOFGRB61gD0dwZw8LSlSNSkm3zcGhAKe95dp2eV3w7S4z21JMlNXWBxG-mjyQ=="
  organization = "iot-lab"
  bucket = "sensores"
```

**Comando para arrancar Telegraf:**

```bash
sudo systemctl start telegraf
sudo systemctl enable telegraf
sudo systemctl status telegraf
```

---

### 🔹 5. InfluxDB — Base de datos de series temporales

InfluxDB recibe los datos procesados desde Telegraf y los almacena en el **bucket "sensores"**.  
Los registros contienen campos como `sonido` y etiquetas de tiempo automáticas.

**Comando para iniciar InfluxDB:**

```bash
sudo systemctl start influxdb
sudo systemctl enable influxdb
sudo systemctl status influxdb
```

---

### 🔹 6. Grafana — Visualización de datos

Grafana se conecta a InfluxDB para mostrar los niveles de sonido en un panel dinámico y actualizable en tiempo real.

**Comandos para ejecutar Grafana:**

```bash
sudo systemctl start grafana-server
sudo systemctl enable grafana-server
sudo systemctl status grafana-server
```

**Visualización Gráfica del sonido en Grafana:**

![Dashboard Sonido](https://raw.githubusercontent.com/Jahziel43/spContent/refs/heads/main/Dashboard%20Sonido.png)

---

## 🚀 Resumen del flujo de ejecución

1. **Ejecutar Mosquitto:**
   - `sudo systemctl start mosquitto`
2. **Ejecutar InfluxDB y Telegraf:**
   - `sudo systemctl start influxdb`
   - `sudo systemctl start telegraf`
3. **Ejecutar Grafana:**
   - `sudo systemctl start grafana-server`
4. **Ejecutar script de lectura/enlace (en Windows):**
   - `python Lector_Microbit_MQTT.py`
5. **Verificar recepción de datos con Mosquitto:**
   - `mosquitto_sub ...`

---

## 🧩 Próximos pasos (por implementar)

- Incorporar control de **relé o LED físico** conectado al servidor o microcontrolador, que se active cuando `sound_lvl > 80`.  
- Integrar la automatización en Grafana mediante **alertas o webhooks**.  
- Evaluar consumo energético y latencia en la comunicación IoT.

---

## 📚 Conclusión

Este sistema demuestra una arquitectura IoT completa, desde la **captura de datos sensoriales** hasta la **visualización en tiempo real** en un dashboard profesional.  
El uso de protocolos estandarizados como MQTT y TLS garantiza **seguridad, eficiencia y escalabilidad**, sentando las bases para expandir el proyecto con automatización física y control remoto en futuras versiones.
