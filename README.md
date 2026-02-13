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

## 🔄 Changelog

### v1.0.0
- ✅ FaceNet com TensorRT
- ✅ Qdrant integration
- ✅ Anti-spoofing
