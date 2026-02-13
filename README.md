# 👤 Face Recognition

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
