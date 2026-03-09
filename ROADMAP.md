# 🗺️ HOJA DE RUTA - Tambo de Camar AR

## 📋 Visión General
Aplicación WebAR que superpone y ancla el modelo 3D del Tambo de Camar sobre una maqueta impresa física. Los usuarios escanean dos códigos QR para conectarse a un servidor local y acceder a la experiencia AR.

**Dispositivos target:** iOS (Safari) + Android (Chrome)
**Servidor:** Notebook local + Router WiFi
**Anclaje:** Image Tracking (reconoce maqueta impresa)
**Contenido:** Modelo 3D con animaciones integradas

---

## 🏗️ FASE 0: PREPARACIÓN (1-2 semanas)

### Subtarea 0.1: Auditoría técnica
- [ ] Exportar modelo comprimido desde app de compresión
- [ ] Verificar dimensiones y peso final del GLB
- [ ] Confirmar animaciones están embebidas en modelo
- [ ] Obtener imagen de alta resolución de la maqueta (para image tracking)
- [ ] Documentar specs: tamaño real maqueta (cm), distancia de captura óptima

### Subtarea 0.2: Seleccionar stack tecnológico
```
Backend:    Node.js + Express
Frontend:   Three.js + WebXR API (o Babylon.js)
Anclaje:    three-ar (Three.js AR) o Babylon.js WebXR
QR:         qrcode.js (generación dinámica)
Compresión: GLB ya comprimido (de app anterior)
Deployment: HTTP local + Service Worker (PWA)
```

### Subtarea 0.3: Configurar entorno de desarrollo
- [ ] Instalar Node.js + npm en notebook
- [ ] Setup repository con estructura modular
- [ ] Instalar dependencias base (Express, vite, three.js)
- [ ] Crear `.gitignore` para modelos/assets grandes

---

## 🎨 FASE 1: ARQUITECTURA Y DISEÑO (1-2 semanas)

### Subtarea 1.1: Diseñar flujo de usuario
```
┌─────────────────────────────────────────────────┐
│ 1. Escanear QR1 (Conexión)                      │
│    └─ Navega a http://192.168.x.x:8080/connect │
│       └─ Detecta IP automáticamente             │
│       └─ Genera sessionID único                 │
│       └─ Muestra QR2 dinámico                   │
│                                                 │
│ 2. Escanear QR2 (Experiencia AR)                │
│    └─ Navega a .../ar?session=ABC123            │
│       └─ Pide permiso de cámara                 │
│       └─ Inicia WebXR                           │
│       └─ Carga modelo local                     │
│       └─ Detecta maqueta impresa                │
│       └─ Ancla modelo sobre maqueta             │
│       └─ Reproducción de animaciones            │
└─────────────────────────────────────────────────┘
```

- [ ] Documentar flujo en Figma/miro (opcional)
- [ ] Definir pantallas: splash → conexión → instrucciones → AR

### Subtarea 1.2: Diseñar estructura de servidor

```
backend/
├── server.js (Express, rutas, WebSocket)
├── routes/
│   ├── connect.js (genera sessionID)
│   ├── ar.js (servir página AR)
│   └── assets.js (streaming de modelos)
├── public/
│   ├── ar.html (página AR con Three.js)
│   ├── connect.html (pantalla de conexión)
│   ├── models/
│   │   └── tambo-camar.glb (comprimido)
│   └── textures/ (si existen)
├── middleware/
│   ├── cors.js
│   └── sessionAuth.js
└── utils/
    ├── qrGenerator.js
    └── imageTracking.js
```

- [ ] Crear estructura de carpetas
- [ ] Documentar rutas API

### Subtarea 1.3: Definir especificaciones de image tracking
- [ ] Dimensión mínima recomendada para la imagen (imprimir en A4)
- [ ] Resolución de la imagen de tracking (≥1024x1024 recomendado)
- [ ] Marcadores visuales (esquinas/bordes con contraste)
- [ ] Distancia óptima de captura (20-150cm)
- [ ] Generar markers de prueba (checkerboard + marca Tambo)

---

## ⚙️ FASE 2: DESARROLLO DEL SERVIDOR (2-3 semanas)

### Subtarea 2.1: Setup básico de Express
```javascript
// server.js
import express from 'express';
import compression from 'compression';
import cors from 'cors';
import { fileURLToPath } from 'url';
import path from 'path';

const app = express();
const __dirname = path.dirname(fileURLToPath(import.meta.url));

app.use(compression());
app.use(cors());
app.use(express.static('public'));
app.use(express.json());

const PORT = process.env.PORT || 8080;
app.listen(PORT, () => console.log(`Servidor en http://localhost:${PORT}`));
```

- [ ] Crear `package.json` con scripts (dev, build, start)
- [ ] Setup Vite para bundling rápido
- [ ] Agregar nodemon para auto-reload en desarrollo

### Subtarea 2.2: Rutas de conexión y sesión
```javascript
// routes/connect.js
GET /connect
  → Retorna HTML con instrucciones
  → Genera sessionID único
  → Devuelve QR2 dinámico (canvas o SVG)

GET /api/sessions/:sessionId
  → Valida sesión activa
  → Retorna IP local + config

POST /api/sessions
  → Crea nueva sesión
  → JSON: { sessionId, qrUrl, expires }
```

- [ ] Implementar generador de sessionID (uuid)
- [ ] Setup generador QR (qrcode.js)
- [ ] Agregar expiración de sesiones (30 min)
- [ ] Logging de sesiones activas

### Subtarea 2.3: Servicio de assets y caché
```javascript
// routes/assets.js
GET /models/tambo-camar.glb
  → Comprime con gzip si es HTTP2
  → Agrega headers de caché (max-age: 86400)
  → Streamea en chunks grandes

GET /config/scene.json
  → Retorna parámetros AR:
    {
      "modelScale": 1.0,
      "modelOffset": { x: 0, y: 0, z: 0 },
      "animationNames": ["anim1", "anim2"],
      "trackingImage": "/images/maqueta-marker.jpg"
    }
```

- [ ] Implementar streaming de GLB
- [ ] Crear config JSON de escena
- [ ] Agregar headers CORS apropiados
- [ ] Setup caching con ETag

### Subtarea 2.4: WebSocket para sincronización
```javascript
// Opcional: sincronizar estado entre múltiples dispositivos
io.on('connection', (socket) => {
  socket.on('ar-ready', (sessionId) => {
    socket.to(sessionId).emit('peer-ready');
  });
  socket.on('animation-play', (data) => {
    socket.to(sessionId).emit('animation-start', data);
  });
});
```

- [ ] Setup socket.io (opcional, para futuras features)
- [ ] Definir eventos de sincronización
- [ ] Documentar API WebSocket

### Subtarea 2.5: Testing del servidor
- [ ] Probar en local: `localhost:8080`
- [ ] Probar IP local: `192.168.x.x:8080`
- [ ] Verificar CORS
- [ ] Verificar compresión de assets
- [ ] Probar desde móvil en red local

---

## 🎬 FASE 3: FRONTEND AR CON THREE.JS (3-4 semanas)

### Subtarea 3.1: Estructura base HTML/CSS
```html
<!-- ar.html -->
<canvas id="ar-canvas"></canvas>
<div id="ui-overlay">
  <div id="loading-screen"></div>
  <div id="ar-instructions"></div>
  <div id="animation-controls"></div>
  <div id="status-indicator"></div>
</div>
```

- [ ] Crear HTML semántico
- [ ] Setup CSS con viewport responsivo
- [ ] Agregar estilos de loader + UI overlay
- [ ] Ensure safe-area-inset para notches

### Subtarea 3.2: Inicializar Three.js + WebXR
```javascript
// ar.js - Core renderer
import * as THREE from 'three';
import { ARButton } from 'three/examples/jsm/webxr/ARButton.js';

const scene = new THREE.Scene();
const camera = new THREE.PerspectiveCamera(...);
const renderer = new THREE.WebGLRenderer({ antialias: true, alpha: true });

document.body.appendChild(ARButton.createButton(renderer, {
  requiredFeatures: ['hit-test'],
  optionalFeatures: ['dom-overlay'],
  domOverlay: { root: document.body }
}));

renderer.xr.enabled = true;
renderer.xr.setReferenceSpaceType('local');
```

- [ ] Setup Three.js scene con lighting
- [ ] Configurar WebXR (hit-test para planos)
- [ ] Agregar ARButton
- [ ] Setup camera perspective para AR
- [ ] Ensure compatible con iOS + Android

### Subtarea 3.3: Cargar modelo GLB con animaciones
```javascript
import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader.js';
import { DRACOLoader } from 'three/examples/jsm/loaders/DRACOLoader.js';

const loader = new GLTFLoader();
const dracoLoader = new DRACOLoader();
dracoLoader.setDecoderPath('/decoders/');
loader.setDRACOLoader(dracoLoader);

loader.load('models/tambo-camar.glb', (gltf) => {
  const model = gltf.scene;
  const animations = gltf.animations;

  scene.add(model);

  // Setup AnimationMixer
  const mixer = new THREE.AnimationMixer(model);
  animations.forEach(clip => {
    mixer.clipAction(clip).clampWhenFinished = true;
  });
});
```

- [ ] Implementar GLTFLoader + DRACOLoader
- [ ] Setup AnimationMixer
- [ ] Crear lista de animaciones disponibles
- [ ] Agregar controles de reproducción (play/pause/speed)
- [ ] Optimizar: usar SkinnedMesh si aplica

### Subtarea 3.4: Image Tracking (anclaje sobre maqueta)
```javascript
// Usar tres-ar para image tracking
import { ImageTracking } from 'three-ar/image-tracking.js';

const tracker = new ImageTracking(renderer);
tracker.loadImage('images/maqueta-marker.jpg');

tracker.onImageFound = (pose) => {
  // Actualizar posición del modelo según maqueta
  model.position.copy(pose.position);
  model.quaternion.copy(pose.quaternion);
};

tracker.onImageLost = () => {
  // Mostrar instrucción para re-encontrar maqueta
};
```

**Alternativas:**
- `three-ar`: Lightweight image tracking
- `MediaPipe`: ML-based detection (más robusto)
- `8th Wall`: Servicio externo (freemium)

- [ ] Seleccionar solución de tracking
- [ ] Implementar image recognition
- [ ] Setup pose estimation
- [ ] Agregar feedback visual (outline/highlight)
- [ ] Optimizar frecuencia de detection (30 FPS target)

### Subtarea 3.5: Controles de animación en AR
```javascript
<div id="animation-controls">
  <button id="play-btn">▶ Play</button>
  <button id="pause-btn">⏸ Pause</button>
  <select id="animation-select">
    <option value="anim1">Animación 1</option>
    <option value="anim2">Animación 2</option>
  </select>
  <input type="range" id="speed-control" min="0.5" max="2" step="0.1" value="1">
</div>
```

- [ ] Crear UI de controles
- [ ] Implementar play/pause/stop
- [ ] Selector de animación
- [ ] Control de velocidad
- [ ] Looping automático (opcional)

### Subtarea 3.6: Gestos y interacción
```javascript
// Touch gestures para interacción en AR
window.addEventListener('touchstart', (e) => {
  if (e.touches.length === 1) {
    // Drag para rotar modelo
  } else if (e.touches.length === 2) {
    // Pinch para escalar modelo
  }
});
```

- [ ] Implementar drag-to-rotate
- [ ] Implementar pinch-to-scale
- [ ] Agregar constraints (min/max scale)
- [ ] Smooth easing en rotación

### Subtarea 3.7: Testing frontend AR
- [ ] Probar en iPhone con Safari (WebXR)
- [ ] Probar en Android con Chrome (WebXR)
- [ ] Verificar image tracking con maqueta real
- [ ] Probar animaciones (play/pause/velocidad)
- [ ] Verificar performance (FPS)
- [ ] Probar sin ubicación de plano (fallback)

---

## 🖼️ FASE 4: IMAGE TRACKING Y ANCLAJE (2-3 semanas)

### Subtarea 4.1: Preparar imagen de maqueta para tracking
- [ ] Fotografiar maqueta en alta resolución (natural lighting)
- [ ] Procesar imagen:
  - Aumentar contraste
  - Reducir ruido
  - Dimensión: 1024x1024 recomendado
- [ ] Generar keypoints/features (autom. por librería)
- [ ] Guardar en `/public/images/maqueta-marker.jpg`

### Subtarea 4.2: Calibrar offset y escala
```javascript
// Ajustar posicionamiento exacto del modelo sobre maqueta
const modelConfig = {
  scale: 1.0,        // Ajustar si modelo es muy grande/pequeño
  position: {
    x: 0.0,          // Centrar horizontalmente
    y: 0.0,          // Ajustar altura (offset vertical)
    z: 0.0           // Profundidad
  },
  rotation: {
    x: 0,            // Pitch
    y: 0,            // Yaw
    z: 0             // Roll
  }
};
```

- [ ] Mapear dimensiones reales de maqueta a modelo 3D
- [ ] Ajustar escala del modelo (regla: 1 unit = 1cm)
- [ ] Probar alignement en 5+ puntos de la maqueta
- [ ] Documentar valores de calibración en JSON

### Subtarea 4.3: Estabilidad de tracking
```javascript
// Filtrado de pose para evitar jitter
class PoseFilter {
  constructor(smoothFactor = 0.7) {
    this.smoothFactor = smoothFactor;
    this.lastPose = null;
  }

  smooth(newPose) {
    if (!this.lastPose) this.lastPose = newPose;
    return {
      position: lerp(this.lastPose.position, newPose.position, this.smoothFactor),
      quaternion: slerp(this.lastPose.quaternion, newPose.quaternion, this.smoothFactor)
    };
  }
}
```

- [ ] Implementar pose smoothing (Kalman filter o LERP)
- [ ] Reducir latency de detección
- [ ] Agregar hysteresis para on/off
- [ ] Probar estabilidad a diferentes distancias

### Subtarea 4.4: Manejo de tracking perdido
- [ ] Mostrar instrucción visual cuando se pierda tracking
- [ ] Freeze modelo en última posición conocida
- [ ] Mostrar hint: "Mantén la maqueta visible"
- [ ] Auto-resume cuando se recupere tracking
- [ ] Timeout: volver a pantalla de conexión (30s sin tracking)

---

## 📱 FASE 5: COMPATIBILIDAD Y OPTIMIZACIÓN (2-3 semanas)

### Subtarea 5.1: Compatibilidad iOS
```javascript
// iOS/Safari específico
if (/iPhone|iPad/.test(navigator.userAgent)) {
  // Usar WebXR si disponible (iOS 17.4+)
  // Fallback a QuickLook si no
  const supportsWebXR = navigator.xr?.isSessionSupported('immersive-ar');
}
```

- [ ] Probar en iPhone 12+ (WebXR)
- [ ] Probar en iPhone 11- (fallback)
- [ ] Verificar permiso de cámara
- [ ] Probar en modo retrato/landscape
- [ ] Verificar notch/safe-area-inset

### Subtarea 5.2: Compatibilidad Android
- [ ] Probar en Chrome 88+
- [ ] Probar en Firefox Reality
- [ ] Verificar ARCore support
- [ ] Probar con diferentes resoluciones
- [ ] Optimizar para devices de gama media

### Subtarea 5.3: Optimización de performance
```javascript
// Renderer optimization
renderer.setPixelRatio(window.devicePixelRatio * 0.75);  // Reducir sobremuestreo
renderer.shadowMap.enabled = false;  // Shadows caras en AR
renderer.antialias = true;

// LOD para modelo si es necesario
const lod = new THREE.LOD();
lod.addLevel(detailedModel, 0);
lod.addLevel(mediumModel, 50);
lod.addLevel(simpleModel, 200);
```

- [ ] Reducir drawcalls (mesh merging)
- [ ] Optimizar texturas (compression, mipmaps)
- [ ] Usar LOD (Level of Detail) si modelo es complejo
- [ ] Desabilitar sombras en AR (caro)
- [ ] Target: 60 FPS en iPhone 12, 30 FPS en Android gama media

### Subtarea 5.4: PWA (Progressive Web App)
```javascript
// service-worker.js - Cache en cliente
self.addEventListener('install', (event) => {
  event.waitUntil(
    caches.open('tambo-ar-v1').then((cache) => {
      return cache.addAll([
        '/',
        '/ar.html',
        '/models/tambo-camar.glb',
        '/images/maqueta-marker.jpg'
      ]);
    })
  );
});
```

- [ ] Implementar Service Worker
- [ ] Cachear assets estáticos
- [ ] Cachear modelo GLB
- [ ] Agregar manifest.json
- [ ] Permiso de instalación en home screen

### Subtarea 5.5: Testing de carga
- [ ] Probar tiempo de carga en red WiFi 5Ghz
- [ ] Probar en WiFi 2.4Ghz (más realista)
- [ ] Medir TTFB (Time to First Byte)
- [ ] Medir Time to Interactive (TTI)
- [ ] Target: <2s en WiFi local

---

## 🔧 FASE 6: REFINAMIENTO Y FEEDBACK (1-2 semanas)

### Subtarea 6.1: UI/UX refinements
- [ ] Pulir instrucciones en pantalla (español claro)
- [ ] Agregar tooltips interactivos
- [ ] Mejorar pantalla de conexión
- [ ] Agregar controles de brillo/contraste para legibilidad
- [ ] Feedback háptico en gestos (vibración)

### Subtarea 6.2: Logging y debugging
```javascript
// Logger para diagnósticos
class ARLogger {
  log(event, data) {
    console.log(`[${event}]`, data);
    // Enviar a server (opcional)
  }
}

logger.log('AR_SESSION_START', { deviceType, screenSize, fps });
logger.log('IMAGE_TRACKING', { detected: true, confidence: 0.95 });
logger.log('ANIMATION_PLAY', { animationName, duration });
```

- [ ] Implementar logging
- [ ] Agregar performance metrics
- [ ] Panel de debug (visible con key)
- [ ] Exportar logs para análisis

### Subtarea 6.3: Testing en dispositivos reales
- [ ] Testing con 5+ usuarios en iPhone
- [ ] Testing con 5+ usuarios en Android
- [ ] Diferentes condiciones de luz
- [ ] Diferentes distancias a la maqueta
- [ ] Diferentes ángulos de observación
- [ ] Recolectar feedback (form o encuesta)

### Subtarea 6.4: Documentación
- [ ] README.md (setup, instrucciones)
- [ ] DEPLOYMENT.md (cómo publicar)
- [ ] TROUBLESHOOTING.md (problemas comunes)
- [ ] API.md (rutas backend)
- [ ] CONFIG.md (parámetros ajustables)

---

## 🚀 FASE 7: DEPLOYMENT (1 semana)

### Subtarea 7.1: Producción en servidor local
```bash
# Script de inicio
npm run build              # Bundlear con Vite
npm run start             # Iniciar servidor

# O con Docker (opcional)
docker build -t tambo-ar .
docker run -p 8080:8080 tambo-ar
```

- [ ] Setup script de inicio automático
- [ ] Configurar reinicio en crash (PM2)
- [ ] Agregar logs persistentes
- [ ] Crear backup automático

### Subtarea 7.2: Distribución de QR
```
QR1: http://[IP-local]:8080/connect
     └─ Imprimible en A4 (tamaño 5x5cm min)
     └─ Ubicar en entrada/recepción

QR2: Generado dinámicamente (post-conexión)
     └─ Mostrar en pantalla/tablet
     └─ O imprimir antes de sesión
```

- [ ] Generar QRs finales
- [ ] Preparar material impreso
- [ ] Crear guía visual para usuarios
- [ ] Setup tableta/pantalla de display (opcional)

### Subtarea 7.3: Monitoreo en producción
- [ ] Dashboard de sesiones activas
- [ ] Alertas de caídas
- [ ] Métricas de uso
- [ ] Tiempo de respuesta
- [ ] Errores más comunes

---

## 📊 TIMELINE ESTIMADO

```
Fase 0 (Prep):           1-2 weeks
Fase 1 (Design):         1-2 weeks
Fase 2 (Backend):        2-3 weeks
Fase 3 (Frontend AR):    3-4 weeks ⭐ MÁS CRÍTICA
Fase 4 (Tracking):       2-3 weeks
Fase 5 (Optimization):   2-3 weeks
Fase 6 (Refinement):     1-2 weeks
Fase 7 (Deployment):     1 week

TOTAL: 13-20 semanas (≈ 3-5 meses)
```

**Hitos críticos:**
- Semana 2: Stack decidido
- Semana 5: Server funcionando
- Semana 8: AR básico en navegador
- Semana 10: Image tracking integrado
- Semana 15: Testing en dispositivos reales
- Semana 20: Production-ready

---

## ⚠️ RIESGOS Y MITIGACIONES

| Riesgo | Probabilidad | Impacto | Mitigation |
|--------|-------------|--------|-----------|
| WebXR no disponible en dispositivo | Baja | Alto | Fallback a model-viewer o QuickLook |
| Image tracking impreciso | Media | Alto | Múltiples markers o pre-tracking |
| Performance pobre en Android gama media | Alta | Medio | LOD, reducir shaders, optimizar texturas |
| Problema de red local inestable | Baja | Alto | Agregar retry logic, timeout handlers |
| Incompatibilidad iOS/Android | Baja | Muy Alto | Testing exhaustivo desde fase 3 |

---

## 🎯 CRITERIOS DE ÉXITO

✅ AR carga en <2s desde QR2
✅ Modelo ancla sobre maqueta con ±2cm de precisión
✅ Animaciones fluidas (60 FPS en iPhone, 30+ en Android)
✅ Tracking estable por 5+ minutos sin jitter
✅ 95%+ uptime del servidor local
✅ Compatible con iOS 15.1+ y Android 10+
✅ Funciona con múltiples sesiones simultáneas

---

## 📚 RECURSOS Y LIBRERÍAS

**Librerías clave:**
- `three.js` - 3D rendering
- `three-ar` o `MediaPipe` - Image tracking
- `qrcode.js` - QR generation
- `express` - Backend framework
- `vite` - Module bundler
- `socket.io` - Real-time (opcional)

**Documentación:**
- [WebXR API Spec](https://www.w3.org/TR/webxr/)
- [Three.js Documentation](https://threejs.org/docs/)
- [MediaPipe Web](https://solutions.mediapipe.dev/web)

---

## 🤝 EQUIPO RECOMENDADO

- **1x Full-stack dev** (Node.js + Three.js)
- **1x AR specialist** (WebXR, tracking)
- **1x QA/Testing** (dispositivos reales)
- **1x DevOps** (servidor, deployment)

O: **1-2 desarrolladores full-stack** con experiencia en WebAR.

