# 👤 Face Recognition

## 🔗 Navegação

**[🏠 AslamSys](https://github.com/AslamSys)** → **[📚 _system](https://github.com/AslamSys/_system)** → **[📂 Segurança (Jetson Orin Nano)](https://github.com/AslamSys/_system/blob/main/hardware/seguranca/README.md)** → **seguranca-face-recognition**

### Containers Relacionados (seguranca)
- [seguranca-brain](https://github.com/AslamSys/seguranca-brain)
- [seguranca-camera-stream-manager](https://github.com/AslamSys/seguranca-camera-stream-manager)
- [seguranca-yolo-detector](https://github.com/AslamSys/seguranca-yolo-detector)
- [seguranca-event-analyzer](https://github.com/AslamSys/seguranca-event-analyzer)
- [seguranca-alert-manager](https://github.com/AslamSys/seguranca-alert-manager)
- [seguranca-video-recorder](https://github.com/AslamSys/seguranca-video-recorder)

---

**Container:** `face-recognition`  
**Ecossistema:** Segurança  
**Hardware:** Jetson Orin Nano  
**Tecnologias:** FaceNet + ArcFace + Qdrant

---

## 📋 Propósito

Reconhecimento facial em tempo real com FaceNet (embeddings 512D), banco vetorial Qdrant e anti-spoofing. Identifica rostos conhecidos < 200ms.

---

## 🎯 Responsabilidades

- ✅ Detectar rostos em frames (MTCNN)
- ✅ Gerar embeddings 512D (FaceNet)
- ✅ Buscar em banco Qdrant (similaridade cosine)
- ✅ Anti-spoofing (liveness detection)
- ✅ Cadastrar novos rostos

---

## 📊 Performance

```yaml
Latency: < 200 ms (detection + recognition)
Accuracy: 99.2% (LFW dataset)
GPU: 512 MB VRAM
Database: Qdrant (1M faces capacity)
```

---

## 🔌 NATS Topics

### Subscribe
- `seguranca.camera.frame` - Frames para análise

### Publish
- `seguranca.face.recognized` - Rosto identificado
- `seguranca.face.unknown` - Rosto desconhecido
- `seguranca.face.spoofing` - Tentativa de falsificação

---

## 🚀 Docker

```yaml
face-recognition:
  image: face-recognition:cuda
  runtime: nvidia
  environment:
    - FACENET_MODEL=/models/facenet_mobilenet.trt
    - QDRANT_URL=http://mordomo-qdrant:6333
    - MIN_CONFIDENCE=0.85
  volumes:
    - ./models:/models
  deploy:
    resources:
      limits:
        memory: 1G
      reservations:
        devices:
          - driver: nvidia
            capabilities: [gpu]
```

---

## 📚 Stack

- **FaceNet MobileNet** - Lightweight (5MB model)
- **MTCNN** - Face detection
- **Qdrant** - Vector database
- **Silent Face Anti-Spoofing** - Liveness check

---

## �️ Catálogo de Identidades Persistente

Todo rosto detectado é gravado no banco vetorial (Qdrant) e no PostgreSQL, mesmo que desconhecido. Rostos desconhecidos recebem um ID automático (`unknown_001`, `unknown_002`...) e **são preservados** até o usuário rotulá-los.

### Fluxo de identidade
```
Rosto detectado
  ↓
  Busca Qdrant (similaridade cosine > 0.85)
  ├─ Match encontrado  → retorna identity_id (ex: "joao", "maria")
  ├─ Match parcial     → retorna identity_id + confidence baixo
  └─ Sem match         → cria "unknown_NNN", salva embedding + thumbnail

Labelamento posterior (via Mordomo ou dashboard):
  "unknown_001 = João"  → atualiza Qdrant + banco
  Todos os eventos históricos de "unknown_001" passam a mostrar "João"
```

### Esquema de identidade no banco
```json
{
  "identity_id": "unknown_001",  // ou "joao" após labelamento
  "display_name": null,           // null=desconhecido, "João"=mapeado
  "first_seen": "2024-04-01T08:30:00Z",
  "last_seen":  "2024-04-15T18:45:00Z",
  "total_appearances": 47,
  "embedding_id": "qdrant_vec_001",
  "thumbnail_path": "/nas/security/faces/unknown_001.jpg",
  "labeled_by": null,             // "user" ou "auto"
  "notes": null
}
```

### NATS Topics (atualizado)

#### Subscribe
- `seguranca.camera.frame` — Frames para análise
- `seguranca.face.label` — Usuário mapeou uma identidade

```javascript
// Usuário rotula unknown via Mordomo ou dashboard
Topic: "seguranca.face.label"
Payload: {
  "identity_id": "unknown_001",
  "display_name": "João",
  "labeled_by": "user"
}
```

#### Publish
```javascript
// Rosto reconhecido (identidade conhecida ou unknown)
Topic: "seguranca.face.recognized"
Payload: {
  "identity_id": "joao",         // ou "unknown_001"
  "display_name": "João",        // ou null
  "confidence": 0.97,
  "camera_id": "cam_1",
  "zone": "porta_entrada",
  "timestamp": "2024-04-15T08:30:00Z",
  "thumbnail_path": "/nas/security/faces/unknown_001.jpg"
}

// Novo rosto desconhecido (primeira vez que aparece)
Topic: "seguranca.face.new_unknown"
Payload: {
  "identity_id": "unknown_042",
  "thumbnail_path": "/nas/security/faces/unknown_042.jpg",
  "camera_id": "cam_1",
  "zone": "porta_entrada"
}
```

> Thumbnails de rostos são pequenos (~50KB) e ficam no NAS indefinidamente até o usuário decidir limpar.

---
## 🚪 Controle de Acesso por Zona

Quando um rosto é reconhecido em uma câmera de zona de acesso controlado (ex: `porta_entrada`), o sistema verifica as permissões da identidade e publica o resultado **antes** de qualquer ação de desbloqueio.

### Permissões no esquema de identidade
```json
{
  "identity_id": "joao",
  "display_name": "João",
  "access": {
    "allowed_zones": ["porta_entrada", "garagem"],
    "schedule": null,           // null = qualquer horário
    "access_level": "resident"  // "resident" | "guest" | "service" | "blocked"
  }
}
```

**Níveis de acesso:**
| Nível | Exemplo | Comportamento |
|---|---|---|
| `resident` | Você, sua esposa | Acesso irrestrito nas zonas permitidas |
| `guest` | Visita com hora marcada | Acesso com horário limitado (`schedule`) |
| `service` | Faxineiro, entregador fixo | Acesso em dias/horários específicos |
| `blocked` | Pessoa indesejada | Nega acesso + alerta imediato |

### Fluxo
```
Rosto reconhecido na "porta_entrada"
  ↓ Verifica identity.access.allowed_zones contém "porta_entrada"?
  ↓ Verifica horário dentro do schedule?
  ├─ ✅ Autorizado  → publica "seguranca.access.granted"
  └─ ❌ Negado      → publica "seguranca.access.denied" + alerta
```

O `mordomo-iot-orchestrator` escuta `seguranca.access.granted` e envia o comando MQTT para a fechadura smart da zona correspondente.

### NATS Topics — Controle de Acesso

#### Subscribe
```javascript
// Configurar permissão via Mordomo ou dashboard
Topic: "seguranca.face.permission.set"
Payload: {
  "identity_id": "joao",
  "access": {
    "allowed_zones": ["porta_entrada", "garagem"],
    "schedule": { "days": ["mon","fri"], "start": "08:00", "end": "18:00" },
    "access_level": "guest"
  }
}
```

#### Publish
```javascript
// Acesso concedido → iot-orchestrator destrava a fechadura
Topic: "seguranca.access.granted"
Payload: {
  "identity_id": "joao",
  "display_name": "João",
  "zone": "porta_entrada",
  "device_id": "fechadura_entrada",
  "timestamp": "2024-04-15T08:30:00Z",
  "confidence": 0.97
}

// Acesso negado → alert-manager notifica
Topic: "seguranca.access.denied"
Payload: {
  "identity_id": "unknown_042",
  "display_name": null,
  "zone": "porta_entrada",
  "reason": "not_authorized",  // "not_authorized" | "outside_schedule" | "blocked" | "low_confidence"
  "timestamp": "2024-04-15T02:15:00Z",
  "thumbnail_path": "/nas/security/faces/unknown_042.jpg"
}
```

### Mapeamento zona → dispositivo IoT
```yaml
# config/zones_access.yaml
zones:
  porta_entrada:
    device_id: "fechadura_entrada"   # ID no iot-state-cache
    mqtt_topic: "home/locks/entrada"
    min_confidence: 0.90             # Threshold mais alto para acesso físico
  garagem:
    device_id: "fechadura_garagem"
    mqtt_topic: "home/locks/garagem"
    min_confidence: 0.85
```

> **Override por comando:** O `mordomo-iot-orchestrator` também aceita `iot.lock.unlock` diretamente do Mordomo (voz, WhatsApp, dashboard), **sem passar por face recognition**. Útil para abrir remotamente para visitas ou prestadores de serviço.

---
## �🔄 Changelog

### v1.0.0
- ✅ FaceNet com TensorRT
- ✅ Qdrant integration
- ✅ Anti-spoofing
