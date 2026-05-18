# HEIMDALL — Sistema de Vigilancia Inteligente con IA
### Proyecto completo para Raspberry Pi 5 · Portfolio-ready · Nivel intermedio–avanzado

> *"El guardian que nunca duerme."*
> Nombres alternativos: **EyePi**, **SentinelEdge**, **GuardianCore**

---

## 1. Objetivo del proyecto

HEIMDALL convierte una Raspberry Pi 5 en un servidor de vigilancia inteligente capaz de detectar personas en tiempo real usando IA en el borde (*edge AI*), grabar clips de vídeo, enviar alertas por Telegram, y ofrecer un dashboard web con historial de eventos, vista en vivo y métricas del sistema.

**Problema real que resuelve:** Las cámaras IP comerciales son caras, envían datos a la nube de terceros, y carecen de personalización. HEIMDALL corre todo localmente: privacidad total, sin suscripciones, extensible.

**Por qué es atractivo para un portafolio:**
- Edge AI con hardware acelerador (Coral USB TPU)
- Arquitectura de microservicios con Docker Compose
- API REST documentada con FastAPI
- Frontend React con WebSockets en tiempo real
- Base de datos relacional + serie temporal
- Seguridad: JWT, bcrypt, HTTPS con Nginx
- Observable: Prometheus + Grafana integrados

---

## 2. Componentes físicos

| Componente | Modelo recomendado | Precio aprox. | Notas |
|---|---|---|---|
| Placa principal | Raspberry Pi 5 8GB | $80 | Necesaria la de 8GB para IA |
| Fuente de alimentación | Pi 5 USB-C 27W oficial | $12 | No escatimes en potencia |
| Tarjeta SD (OS) | SanDisk Extreme 32GB A2 | $12 | Solo para el OS |
| Almacenamiento | NVMe M.2 512GB + HAT+ | $55 | Clips y base de datos |
| Cámara | Raspberry Pi Camera Module 3 | $25 | 12MP, autofoco, HDR |
| Cable cámara | Cable FPC 22-pin 300mm | $5 | Para Rpi5 (diferente al 4) |
| Sensor movimiento | HC-SR501 PIR | $3 | Activación por presencia |
| Acelerador IA | Google Coral USB Accelerator | $60 | Inferencia YOLO a 30+ FPS |
| Carcasa | Argon ONE V3 M.2 | $35 | Ventilación activa + M.2 slot |
| Cables GPIO | Dupont hembra-hembra 20cm | $4 | Para el PIR |
| Resistencia | 10kΩ pull-down | $0.10 | Para estabilizar GPIO |
| **Total estimado** | | **~$291** | |

**Opcional:** Segundo módulo de cámara (CSI-2 port B), buzzer activo 5V, LED de estado RGB.

---

## 3. Conexiones de hardware

### 3.1 Sensor PIR HC-SR501 → GPIO

```
HC-SR501          Raspberry Pi 5
─────────         ──────────────
VCC     ──────→   Pin 2  (5V)
GND     ──────→   Pin 6  (GND)
OUT     ──────→   Pin 11 (GPIO 17)  ← con resistencia 10kΩ a GND
```

**Configuración del HC-SR501:**
- Jumper: modo repetición (H)
- Potenciómetro sensibilidad: posición central (3–4 metros)
- Potenciómetro tiempo: mínimo (≈3 segundos)

### 3.2 Cámara → Puerto CSI

El Pi Camera Module 3 usa el conector CSI-2 de 22 pines exclusivo del Pi 5.

```bash
# Verificar que la cámara es reconocida
libcamera-hello --list-cameras
```

### 3.3 Coral USB Accelerator

Conectar al puerto USB 3.0 azul (mayor ancho de banda). Verificar:

```bash
lsusb | grep Google
# Debe mostrar: Google Inc. Coral USB Accelerator
```

### 3.4 NVMe SSD (HAT+)

El HAT+ M.2 de Raspberry Pi se conecta directamente sobre los pines GPIO y el conector PCIe gen 2 del Pi 5. No requiere cables adicionales. Montar después de instalar el OS.

---

## 4. Instalación y configuración del sistema operativo

### 4.1 Flashear Raspberry Pi OS Lite 64-bit

```bash
# En tu PC: usar Raspberry Pi Imager
# Imagen: Raspberry Pi OS Lite (64-bit) — Bookworm
# Configuración avanzada (⚙):
#   hostname: heimdall
#   usuario: admin / contraseña_segura
#   SSH: habilitado
#   WiFi: configurar si no usas Ethernet
```

### 4.2 Primera configuración

```bash
# Conectar por SSH
ssh admin@heimdall.local

# Actualizar sistema
sudo apt update && sudo apt full-upgrade -y

# Habilitar la cámara y GPIO
sudo raspi-config
# → Interface Options → Camera → Enable
# → Interface Options → I2C → Enable
# → Advanced → GL Driver → Fake KMS

# Instalar dependencias base
sudo apt install -y git curl vim htop \
  libcamera-dev libcamera-apps \
  python3-picamera2 python3-pip \
  ffmpeg v4l-utils

# Montar el NVMe SSD
sudo fdisk /dev/nvme0n1  # crear partición p1
sudo mkfs.ext4 /dev/nvme0n1p1
sudo mkdir -p /media/heimdall-data
echo '/dev/nvme0n1p1 /media/heimdall-data ext4 defaults,noatime 0 2' | sudo tee -a /etc/fstab
sudo mount -a

# Verificar
df -h | grep heimdall
```

### 4.3 Instalar Docker y Docker Compose

```bash
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker admin
newgrp docker

# Verificar
docker --version
docker compose version
```

### 4.4 Instalar dependencias de Python (Coral TPU)

```bash
echo "deb https://packages.cloud.google.com/apt coral-edgetpu-stable main" \
  | sudo tee /etc/apt/sources.list.d/coral-edgetpu.list
curl https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
sudo apt update
sudo apt install -y libedgetpu1-std python3-pycoral
```

---

## 5. Configuración de red y acceso remoto

### 5.1 IP estática

```bash
# /etc/dhcpcd.conf
interface eth0
static ip_address=192.168.1.100/24
static routers=192.168.1.1
static domain_name_servers=1.1.1.1 8.8.8.8
```

### 5.2 Tailscale VPN (acceso remoto seguro)

```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
# Seguir el enlace para autenticar
# Desde cualquier lugar del mundo: ssh admin@heimdall.tailnet.ts.net
```

### 5.3 Nginx como reverse proxy (HTTPS)

```bash
sudo apt install -y nginx certbot python3-certbot-nginx
# Para dominio propio:
sudo certbot --nginx -d tudominio.duckdns.org
# Para red local (self-signed):
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /etc/ssl/private/heimdall.key \
  -out /etc/ssl/certs/heimdall.crt
```

---

## 6. Estructura del proyecto

```
heimdall/
├── docker-compose.yml
├── .env                          ← secretos (no commitear)
├── nginx/
│   ├── nginx.conf
│   └── certs/
├── services/
│   ├── api-gateway/              ← FastAPI principal
│   │   ├── Dockerfile
│   │   ├── main.py
│   │   ├── routers/
│   │   │   ├── events.py
│   │   │   ├── streams.py
│   │   │   └── auth.py
│   │   ├── models/
│   │   │   └── event.py
│   │   ├── dependencies.py
│   │   └── requirements.txt
│   ├── vision-service/           ← IA + cámara + PIR
│   │   ├── Dockerfile
│   │   ├── detector.py
│   │   ├── camera.py
│   │   ├── pir_listener.py
│   │   ├── models/
│   │   │   └── yolov8n_edgetpu.tflite
│   │   └── requirements.txt
│   ├── notification-service/     ← Telegram + SMTP
│   │   ├── Dockerfile
│   │   ├── notifier.py
│   │   └── requirements.txt
│   ├── recording-service/        ← FFmpeg wrapper
│   │   ├── Dockerfile
│   │   ├── recorder.py
│   │   └── requirements.txt
│   └── stream-service/           ← HLS + WebRTC
│       ├── Dockerfile
│       └── streamer.py
├── frontend/                     ← React + Vite
│   ├── Dockerfile
│   ├── package.json
│   ├── vite.config.js
│   └── src/
│       ├── App.jsx
│       ├── pages/
│       │   ├── Dashboard.jsx
│       │   ├── Events.jsx
│       │   └── Live.jsx
│       └── components/
│           ├── EventCard.jsx
│           ├── LiveFeed.jsx
│           └── MetricsPanel.jsx
├── db/
│   └── init.sql
├── monitoring/
│   ├── prometheus.yml
│   └── grafana/
│       └── dashboards/
└── scripts/
    ├── setup.sh
    ├── backup.sh
    └── update.sh
```

---

## 7. Docker Compose completo

```yaml
# docker-compose.yml
version: '3.9'

services:

  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      - ./nginx/certs:/etc/nginx/certs:ro
    depends_on:
      - api-gateway
      - frontend
    restart: unless-stopped

  api-gateway:
    build: ./services/api-gateway
    environment:
      - DATABASE_URL=postgresql://heimdall:${DB_PASSWORD}@postgres:5432/heimdall
      - REDIS_URL=redis://redis:6379
      - JWT_SECRET=${JWT_SECRET}
    depends_on:
      - postgres
      - redis
    expose:
      - "8000"
    restart: unless-stopped

  vision-service:
    build: ./services/vision-service
    privileged: true                        # acceso a /dev/video0 y GPIO
    devices:
      - /dev/video0:/dev/video0             # cámara v4l2
      - /dev/bus/usb:/dev/bus/usb           # Coral USB
    volumes:
      - /media/heimdall-data:/data
      - /proc/device-tree:/proc/device-tree:ro
    environment:
      - REDIS_URL=redis://redis:6379
      - CONFIDENCE_THRESHOLD=0.72
      - PIR_GPIO_PIN=17
    depends_on:
      - redis
    restart: unless-stopped

  notification-service:
    build: ./services/notification-service
    environment:
      - REDIS_URL=redis://redis:6379
      - TELEGRAM_TOKEN=${TELEGRAM_TOKEN}
      - TELEGRAM_CHAT_ID=${TELEGRAM_CHAT_ID}
      - SMTP_HOST=${SMTP_HOST}
      - SMTP_USER=${SMTP_USER}
      - SMTP_PASS=${SMTP_PASS}
    depends_on:
      - redis
    restart: unless-stopped

  recording-service:
    build: ./services/recording-service
    devices:
      - /dev/video0:/dev/video0
    volumes:
      - /media/heimdall-data/clips:/clips
    environment:
      - REDIS_URL=redis://redis:6379
      - CLIP_DURATION=30
    depends_on:
      - redis
    restart: unless-stopped

  stream-service:
    build: ./services/stream-service
    devices:
      - /dev/video0:/dev/video0
    expose:
      - "8554"          # RTSP
      - "8888"          # HLS
    restart: unless-stopped

  frontend:
    build: ./frontend
    expose:
      - "3000"
    environment:
      - VITE_API_URL=https://heimdall.local/api
      - VITE_WS_URL=wss://heimdall.local/ws
    restart: unless-stopped

  postgres:
    image: postgres:16-alpine
    environment:
      - POSTGRES_DB=heimdall
      - POSTGRES_USER=heimdall
      - POSTGRES_PASSWORD=${DB_PASSWORD}
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./db/init.sql:/docker-entrypoint-initdb.d/init.sql
    restart: unless-stopped

  redis:
    image: redis:7-alpine
    command: redis-server --maxmemory 256mb --maxmemory-policy allkeys-lru
    volumes:
      - redis_data:/data
    restart: unless-stopped

  prometheus:
    image: prom/prometheus:latest
    volumes:
      - ./monitoring/prometheus.yml:/etc/prometheus/prometheus.yml:ro
    expose:
      - "9090"
    restart: unless-stopped

  grafana:
    image: grafana/grafana:latest
    ports:
      - "3001:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=${GRAFANA_PASSWORD}
    volumes:
      - grafana_data:/var/lib/grafana
      - ./monitoring/grafana/dashboards:/etc/grafana/provisioning/dashboards
    depends_on:
      - prometheus
    restart: unless-stopped

volumes:
  postgres_data:
  redis_data:
  grafana_data:
```

---

## 8. Código funcional — componentes clave

### 8.1 Vision Service — detector.py

```python
# services/vision-service/detector.py
"""
Servicio principal de visión: escucha el PIR, captura frames,
ejecuta YOLOv8 en Coral TPU y publica eventos en Redis.
"""
import time
import asyncio
import logging
import numpy as np
import redis.asyncio as aioredis
from picamera2 import Picamera2
from pycoral.adapters import common, detect
from pycoral.utils.dataset import read_label_file
from pycoral.utils.edgetpu import make_interpreter
import RPi.GPIO as GPIO

logging.basicConfig(level=logging.INFO, format='%(asctime)s [%(levelname)s] %(message)s')
log = logging.getLogger("vision")

PIR_PIN       = 17
MODEL_PATH    = "/models/yolov8n_edgetpu.tflite"
LABELS_PATH   = "/models/coco_labels.txt"
CONFIDENCE    = 0.72
REDIS_URL     = "redis://redis:6379"
COOLDOWN_SEC  = 10          # evitar alertas duplicadas

# ── GPIO ──────────────────────────────────────────────────────────
GPIO.setmode(GPIO.BCM)
GPIO.setup(PIR_PIN, GPIO.IN, pull_up_down=GPIO.PUD_DOWN)

# ── Coral TPU ─────────────────────────────────────────────────────
interpreter = make_interpreter(MODEL_PATH)
interpreter.allocate_tensors()
labels = read_label_file(LABELS_PATH)
input_size = common.input_size(interpreter)   # (300, 300) o (640, 640)
log.info(f"Modelo cargado: input {input_size}")

# ── Cámara ────────────────────────────────────────────────────────
cam = Picamera2()
config = cam.create_still_configuration(
    main={"size": (1920, 1080), "format": "RGB888"},
    lores={"size": input_size, "format": "RGB888"},
)
cam.configure(config)
cam.start()
log.info("Cámara iniciada")

async def publish_event(redis: aioredis.Redis, detections: list, frame_path: str):
    import json, datetime
    payload = {
        "timestamp": datetime.datetime.utcnow().isoformat(),
        "detections": detections,
        "snapshot_path": frame_path,
        "confidence_max": max(d["confidence"] for d in detections),
    }
    await redis.publish("motion_detected", json.dumps(payload))
    log.info(f"Evento publicado: {len(detections)} persona(s) detectada(s)")

def run_inference(frame_lores: np.ndarray) -> list:
    """Retorna lista de detecciones de personas con confianza >= CONFIDENCE."""
    common.set_input(interpreter, frame_lores)
    interpreter.invoke()
    objs = detect.get_objects(interpreter, score_threshold=CONFIDENCE)
    persons = [
        {"label": labels.get(obj.id, "unknown"), "confidence": float(obj.score),
         "bbox": [obj.bbox.xmin, obj.bbox.ymin, obj.bbox.xmax, obj.bbox.ymax]}
        for obj in objs if labels.get(obj.id) == "person"
    ]
    return persons

def save_snapshot(frame_hires: np.ndarray) -> str:
    import cv2, datetime, os
    ts = datetime.datetime.utcnow().strftime("%Y%m%d_%H%M%S")
    path = f"/data/snapshots/{ts}.jpg"
    os.makedirs("/data/snapshots", exist_ok=True)
    cv2.imwrite(path, cv2.cvtColor(frame_hires, cv2.COLOR_RGB2BGR), [cv2.IMWRITE_JPEG_QUALITY, 92])
    return path

async def main():
    redis = await aioredis.from_url(REDIS_URL, decode_responses=True)
    last_alert = 0
    log.info("Esperando movimiento...")

    loop = asyncio.get_event_loop()

    def pir_callback(channel):
        asyncio.run_coroutine_threadsafe(handle_motion(redis), loop)

    GPIO.add_event_detect(PIR_PIN, GPIO.RISING, callback=pir_callback, bouncetime=2000)

    try:
        while True:
            await asyncio.sleep(1)
    except KeyboardInterrupt:
        pass
    finally:
        cam.stop()
        GPIO.cleanup()

async def handle_motion(redis: aioredis.Redis):
    global last_alert
    now = time.time()
    if now - last_alert < COOLDOWN_SEC:
        return

    log.info("PIR activado — capturando frame")
    frame = cam.capture_array("main")
    frame_lores = cam.capture_array("lores")

    detections = run_inference(frame_lores)

    if detections:
        snapshot_path = save_snapshot(frame)
        await publish_event(redis, detections, snapshot_path)
        last_alert = now
    else:
        log.info("Movimiento sin personas (planta, mascota, etc.)")

if __name__ == "__main__":
    asyncio.run(main())
```

### 8.2 API Gateway — main.py

```python
# services/api-gateway/main.py
from fastapi import FastAPI, WebSocket, Depends, HTTPException, status
from fastapi.middleware.cors import CORSMiddleware
from fastapi.security import OAuth2PasswordBearer, OAuth2PasswordRequestForm
from contextlib import asynccontextmanager
import redis.asyncio as aioredis
import asyncpg
import json, os, asyncio, logging

from routers import events, streams, auth

log = logging.getLogger("api-gateway")

@asynccontextmanager
async def lifespan(app: FastAPI):
    app.state.redis = await aioredis.from_url(os.environ["REDIS_URL"], decode_responses=True)
    app.state.db    = await asyncpg.create_pool(os.environ["DATABASE_URL"])
    log.info("Conexiones establecidas")
    yield
    await app.state.redis.close()
    await app.state.db.close()

app = FastAPI(
    title="HEIMDALL API",
    version="1.0.0",
    description="Sistema de vigilancia inteligente — API REST",
    lifespan=lifespan,
)

app.add_middleware(
    CORSMiddleware,
    allow_origins=["https://heimdall.local"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

app.include_router(auth.router,    prefix="/api/auth",    tags=["auth"])
app.include_router(events.router,  prefix="/api/events",  tags=["events"])
app.include_router(streams.router, prefix="/api/streams", tags=["streams"])

# WebSocket para dashboard en tiempo real
@app.websocket("/ws")
async def websocket_events(ws: WebSocket):
    await ws.accept()
    redis: aioredis.Redis = ws.app.state.redis
    pubsub = redis.pubsub()
    await pubsub.subscribe("motion_detected", "recording_saved")
    try:
        async for message in pubsub.listen():
            if message["type"] == "message":
                await ws.send_text(message["data"])
    except Exception:
        pass
    finally:
        await pubsub.unsubscribe()

@app.get("/health")
async def health():
    return {"status": "ok", "service": "heimdall-api"}
```

### 8.3 events.py — Router de eventos

```python
# services/api-gateway/routers/events.py
from fastapi import APIRouter, Depends, Query
from typing import Optional
import asyncpg
from dependencies import get_db, require_auth

router = APIRouter()

@router.get("/")
async def list_events(
    page: int = Query(1, ge=1),
    limit: int = Query(20, le=100),
    from_date: Optional[str] = None,
    db: asyncpg.Pool = Depends(get_db),
    _: dict = Depends(require_auth),
):
    offset = (page - 1) * limit
    where = "WHERE timestamp >= $3" if from_date else ""
    params = [limit, offset] + ([from_date] if from_date else [])

    rows = await db.fetch(f"""
        SELECT id, timestamp, confidence_max, snapshot_path,
               detection_count, clip_path
        FROM events
        {where}
        ORDER BY timestamp DESC
        LIMIT $1 OFFSET $2
    """, *params)

    total = await db.fetchval(f"SELECT COUNT(*) FROM events {where}",
                              *([from_date] if from_date else []))
    return {
        "data": [dict(r) for r in rows],
        "pagination": {"page": page, "limit": limit, "total": total},
    }

@router.get("/{event_id}")
async def get_event(
    event_id: int,
    db: asyncpg.Pool = Depends(get_db),
    _: dict = Depends(require_auth),
):
    row = await db.fetchrow("SELECT * FROM events WHERE id = $1", event_id)
    if not row:
        from fastapi import HTTPException
        raise HTTPException(status_code=404, detail="Evento no encontrado")
    return dict(row)
```

### 8.4 Notification Service

```python
# services/notification-service/notifier.py
"""
Escucha el canal Redis 'motion_detected' y envía alertas
por Telegram (con foto) y SMTP como backup.
"""
import asyncio, json, os, logging, aiohttp, aiofiles, smtplib
from email.mime.multipart import MIMEMultipart
from email.mime.text import MIMEText
from email.mime.image import MIMEImage
import redis.asyncio as aioredis

log = logging.getLogger("notifier")

TELEGRAM_TOKEN   = os.environ["TELEGRAM_TOKEN"]
TELEGRAM_CHAT_ID = os.environ["TELEGRAM_CHAT_ID"]
SMTP_HOST        = os.environ.get("SMTP_HOST", "smtp.gmail.com")
SMTP_USER        = os.environ.get("SMTP_USER", "")
SMTP_PASS        = os.environ.get("SMTP_PASS", "")

async def send_telegram(session: aiohttp.ClientSession, snapshot_path: str, event: dict):
    caption = (
        f"🚨 *HEIMDALL — Intruso detectado*\n"
        f"⏰ `{event['timestamp']}`\n"
        f"👤 Personas: {len(event['detections'])}\n"
        f"📊 Confianza: `{event['confidence_max']*100:.1f}%`"
    )
    url = f"https://api.telegram.org/bot{TELEGRAM_TOKEN}/sendPhoto"
    async with aiofiles.open(snapshot_path, "rb") as f:
        photo_bytes = await f.read()

    data = aiohttp.FormData()
    data.add_field("chat_id", TELEGRAM_CHAT_ID)
    data.add_field("caption", caption)
    data.add_field("parse_mode", "Markdown")
    data.add_field("photo", photo_bytes, filename="snapshot.jpg", content_type="image/jpeg")

    async with session.post(url, data=data) as resp:
        if resp.status == 200:
            log.info("Alerta Telegram enviada")
        else:
            log.error(f"Telegram error: {await resp.text()}")

async def main():
    redis = await aioredis.from_url(os.environ["REDIS_URL"], decode_responses=True)
    pubsub = redis.pubsub()
    await pubsub.subscribe("motion_detected")
    log.info("Escuchando canal motion_detected...")

    async with aiohttp.ClientSession() as session:
        async for message in pubsub.listen():
            if message["type"] != "message":
                continue
            event = json.loads(message["data"])
            try:
                await send_telegram(session, event["snapshot_path"], event)
            except Exception as e:
                log.error(f"Error notificando: {e}")

if __name__ == "__main__":
    asyncio.run(main())
```

### 8.5 Base de datos — init.sql

```sql
-- db/init.sql
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

CREATE TABLE events (
    id               SERIAL PRIMARY KEY,
    uuid             UUID DEFAULT uuid_generate_v4() UNIQUE,
    timestamp        TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    confidence_max   REAL NOT NULL,
    detection_count  INTEGER NOT NULL DEFAULT 1,
    snapshot_path    TEXT,
    clip_path        TEXT,
    alert_sent       BOOLEAN DEFAULT FALSE,
    metadata         JSONB DEFAULT '{}'
);

CREATE INDEX idx_events_timestamp ON events (timestamp DESC);
CREATE INDEX idx_events_confidence ON events (confidence_max);

CREATE TABLE users (
    id           SERIAL PRIMARY KEY,
    username     VARCHAR(50) UNIQUE NOT NULL,
    password_hash TEXT NOT NULL,
    role         VARCHAR(20) DEFAULT 'viewer',
    created_at   TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE system_config (
    key   VARCHAR(100) PRIMARY KEY,
    value TEXT NOT NULL,
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

INSERT INTO system_config VALUES
    ('pir_enabled',          'true',  NOW()),
    ('confidence_threshold', '0.72',  NOW()),
    ('clip_duration_sec',    '30',    NOW()),
    ('cooldown_sec',         '10',    NOW()),
    ('telegram_enabled',     'true',  NOW());

-- Vista: resumen de actividad por hora
CREATE VIEW hourly_activity AS
    SELECT DATE_TRUNC('hour', timestamp) AS hour,
           COUNT(*) AS event_count,
           AVG(confidence_max) AS avg_confidence
    FROM events
    GROUP BY 1
    ORDER BY 1 DESC;
```

### 8.6 Frontend React — Dashboard.jsx

```jsx
// frontend/src/pages/Dashboard.jsx
import { useState, useEffect, useRef } from "react";

const WS_URL = import.meta.env.VITE_WS_URL || "ws://localhost:8000/ws";
const API    = import.meta.env.VITE_API_URL || "http://localhost:8000/api";

export default function Dashboard() {
  const [events, setEvents]     = useState([]);
  const [stats, setStats]       = useState({ today: 0, hour: 0, confidence: 0 });
  const [connected, setConnected] = useState(false);
  const ws = useRef(null);

  // Cargar eventos históricos
  useEffect(() => {
    const token = localStorage.getItem("heimdall_token");
    fetch(`${API}/events?limit=20`, {
      headers: { Authorization: `Bearer ${token}` }
    })
      .then(r => r.json())
      .then(data => setEvents(data.data || []));
  }, []);

  // WebSocket en tiempo real
  useEffect(() => {
    const connect = () => {
      ws.current = new WebSocket(WS_URL);
      ws.current.onopen  = () => setConnected(true);
      ws.current.onclose = () => {
        setConnected(false);
        setTimeout(connect, 3000);    // reconectar automáticamente
      };
      ws.current.onmessage = (e) => {
        const event = JSON.parse(e.data);
        if (event.timestamp) {
          setEvents(prev => [event, ...prev.slice(0, 49)]);   // max 50
          setStats(prev => ({ ...prev, today: prev.today + 1 }));
        }
      };
    };
    connect();
    return () => ws.current?.close();
  }, []);

  return (
    <div style={{ padding: "2rem", fontFamily: "system-ui" }}>
      {/* Header */}
      <div style={{ display: "flex", justifyContent: "space-between", alignItems: "center", marginBottom: "1.5rem" }}>
        <h1 style={{ margin: 0, fontSize: "1.8rem" }}>⚡ HEIMDALL</h1>
        <span style={{
          padding: "4px 12px", borderRadius: "20px", fontSize: "13px",
          background: connected ? "#d1fae5" : "#fee2e2",
          color: connected ? "#065f46" : "#991b1b"
        }}>
          {connected ? "● Conectado" : "○ Reconectando..."}
        </span>
      </div>

      {/* Stats */}
      <div style={{ display: "grid", gridTemplateColumns: "repeat(3, 1fr)", gap: "1rem", marginBottom: "2rem" }}>
        {[
          { label: "Eventos hoy",     value: stats.today },
          { label: "Última hora",     value: stats.hour },
          { label: "Confianza media", value: `${(stats.confidence * 100).toFixed(0)}%` },
        ].map(s => (
          <div key={s.label} style={{ background: "#f8fafc", borderRadius: "12px", padding: "1.25rem", border: "1px solid #e2e8f0" }}>
            <div style={{ fontSize: "13px", color: "#64748b", marginBottom: "4px" }}>{s.label}</div>
            <div style={{ fontSize: "2rem", fontWeight: "600" }}>{s.value}</div>
          </div>
        ))}
      </div>

      {/* Eventos recientes */}
      <h2 style={{ fontSize: "1.1rem", marginBottom: "1rem" }}>Eventos recientes</h2>
      <div style={{ display: "flex", flexDirection: "column", gap: "0.75rem" }}>
        {events.map((ev, i) => (
          <div key={i} style={{
            display: "flex", alignItems: "center", gap: "1rem",
            padding: "0.875rem 1rem", borderRadius: "10px",
            background: "#fff", border: "1px solid #e2e8f0"
          }}>
            {ev.snapshot_path && (
              <img
                src={`${API}/media/${ev.snapshot_path.split("/").pop()}`}
                alt="snapshot"
                style={{ width: 80, height: 50, objectFit: "cover", borderRadius: "6px" }}
              />
            )}
            <div style={{ flex: 1 }}>
              <div style={{ fontWeight: "500", fontSize: "14px" }}>
                👤 {ev.detections?.length || 1} persona(s) detectada(s)
              </div>
              <div style={{ color: "#94a3b8", fontSize: "12px", marginTop: "2px" }}>
                {new Date(ev.timestamp).toLocaleString("es-ES")}
              </div>
            </div>
            <span style={{
              padding: "3px 10px", borderRadius: "6px", fontSize: "12px",
              background: "#eff6ff", color: "#1d4ed8", fontWeight: "500"
            }}>
              {((ev.confidence_max || 0.8) * 100).toFixed(0)}%
            </span>
          </div>
        ))}
      </div>
    </div>
  );
}
```

---

## 9. Variables de entorno (.env)

```bash
# .env — NO commitear, agregar al .gitignore
DB_PASSWORD=cambia_esta_contrasena_db_123!
JWT_SECRET=genera_esto_con_openssl_rand_hex_32
TELEGRAM_TOKEN=123456789:AAFxxx...
TELEGRAM_CHAT_ID=-100xxxxxxxx
SMTP_HOST=smtp.gmail.com
SMTP_USER=tu_correo@gmail.com
SMTP_PASS=tu_app_password_gmail
GRAFANA_PASSWORD=admin_seguro_456!
```

---

## 10. Cómo ejecutar el proyecto

```bash
# 1. Clonar en la Raspberry Pi
git clone https://github.com/tu-usuario/heimdall.git
cd heimdall

# 2. Configurar secretos
cp .env.example .env
nano .env   # editar con tus valores reales

# 3. Construir y levantar todos los servicios
docker compose up -d --build

# 4. Ver logs en tiempo real
docker compose logs -f vision-service

# 5. Verificar que todo está corriendo
docker compose ps

# 6. Acceder al dashboard
# → https://heimdall.local          (frontend React)
# → https://heimdall.local/api/docs (FastAPI Swagger)
# → http://heimdall.local:3001      (Grafana)

# 7. Primer usuario admin
docker compose exec api-gateway python -c "
from dependencies import create_user
create_user('admin', 'contrasena_segura', role='admin')
"
```

---

## 11. Medidas de seguridad

| Capa | Medida | Implementación |
|---|---|---|
| Red | HTTPS everywhere | Nginx + self-signed / Let's Encrypt |
| Red | VPN acceso remoto | Tailscale (wireguard) |
| API | Autenticación | JWT Bearer tokens, 24h expiry |
| API | Autorización | RBAC: admin / viewer |
| API | Rate limiting | FastAPI-limiter + Redis |
| Contraseñas | Hash seguro | bcrypt cost=12 |
| Contenedores | Usuario no-root | USER 1000:1000 en Dockerfiles |
| Secretos | Sin hardcodear | Variables de entorno + .gitignore |
| Firewall | Solo puertos necesarios | `ufw allow 80,443,22,41641/udp` |
| Actualizaciones | Dependabot | `docker compose pull && up -d` |

```bash
# Configurar firewall básico
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow ssh
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw allow 41641/udp    # Tailscale
sudo ufw enable
```

---

## 12. Optimizaciones para Raspberry Pi 5

```bash
# GPU memory (la IA usa CPU+TPU, no GPU)
echo "gpu_mem=128" | sudo tee -a /boot/firmware/config.txt

# Overclock moderado (con disipador activo)
echo "[pi5]
arm_freq=2800
over_voltage=2" | sudo tee -a /boot/firmware/config.txt

# Swap en SSD (no en SD)
sudo dphys-swapfile swapoff
sudo sed -i 's/CONF_SWAPFILE=.*/CONF_SWAPFILE=\/media\/heimdall-data\/swapfile/' /etc/dphys-swapfile
sudo sed -i 's/CONF_SWAPSIZE=.*/CONF_SWAPSIZE=2048/' /etc/dphys-swapfile
sudo dphys-swapfile setup && sudo dphys-swapfile swapon

# Desactivar servicios innecesarios
sudo systemctl disable bluetooth avahi-daemon
sudo systemctl disable triggerhappy cups

# Límites de recursos por contenedor en docker-compose.yml
# Agregar bajo cada service:
deploy:
  resources:
    limits:
      memory: 512M
    reservations:
      memory: 256M
```

---

## 13. Despliegue en producción

```bash
# Crear servicio systemd para auto-inicio
sudo tee /etc/systemd/system/heimdall.service << 'EOF'
[Unit]
Description=HEIMDALL Surveillance System
Requires=docker.service
After=docker.service network-online.target

[Service]
Type=oneshot
RemainAfterExit=yes
WorkingDirectory=/home/admin/heimdall
ExecStart=/usr/bin/docker compose up -d
ExecStop=/usr/bin/docker compose down
TimeoutStartSec=300
User=admin

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable heimdall
sudo systemctl start heimdall

# Backup automático (crontab)
(crontab -l; echo "0 3 * * * /home/admin/heimdall/scripts/backup.sh") | crontab -

# scripts/backup.sh
#!/bin/bash
DATE=$(date +%Y%m%d)
BACKUP_DIR="/media/heimdall-data/backups"
mkdir -p $BACKUP_DIR
docker exec heimdall-postgres-1 \
  pg_dump -U heimdall heimdall | gzip > "$BACKUP_DIR/db_$DATE.sql.gz"
find $BACKUP_DIR -name "db_*.sql.gz" -mtime +30 -delete
echo "Backup completado: $DATE"
```

---

## 14. Mejoras futuras

| Prioridad | Mejora | Tecnología |
|---|---|---|
| Alta | Reconocimiento facial para lista blanca/negra | face_recognition + DeepFace |
| Alta | Detección de zona — alertar solo en áreas específicas | OpenCV ROI masking |
| Media | App móvil con notificaciones push | React Native + Expo |
| Media | Múltiples cámaras (Pi Camera B + USB cams) | MultiProcessing + v4l2 |
| Media | Análisis de patrones — horarios de mayor actividad | TimescaleDB + Chart.js |
| Media | Reconocimiento de placas vehiculares | PaddleOCR + OpenCV |
| Baja | Integración con Home Assistant | HA MQTT integration |
| Baja | Almacenamiento en S3 / Backblaze B2 | boto3 / rclone |
| Baja | Detección de audio (cristales rotos, alarmas) | PyAudio + TensorFlow Lite |
| Investigación | LLM local para descripción de escenas | llama.cpp + LLaVA |

---

## 15. Por qué HEIMDALL es un proyecto de portafolio excepcional

1. **Problema real** — privacidad, seguridad doméstica, IoT industrial
2. **Stack moderno completo** — Python async, React, PostgreSQL, Redis, Docker
3. **IA en producción** — no es un notebook de Colab, es IA desplegada en hardware real
4. **Microservicios** — arquitectura escalable y desacoplada
5. **Observable** — Prometheus + Grafana, logs estructurados
6. **Seguro** — JWT, HTTPS, bcrypt, firewall, VPN
7. **CI-ready** — estructura de proyecto clara, Dockerfiles reproducibles
8. **Extensible** — cada servicio puede crecer independientemente

> Este proyecto demuestra comprensión de sistemas distribuidos, edge computing, seguridad de APIs y desarrollo full-stack — exactamente lo que buscan los empleadores de empresas de software modernas.
