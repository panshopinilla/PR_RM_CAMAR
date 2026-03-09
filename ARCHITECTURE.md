# 🏛️ ARQUITECTURA TÉCNICA - Tambo de Camar AR

## 1. STACK SELECCIONADO

### Backend
```
Node.js 18+ LTS
├── Express 4.x (web framework)
├── Vite (bundler, dev server)
├── uuid (session management)
├── qrcode (QR generation)
└── compression (gzip middleware)
```

### Frontend
```
HTML5 / CSS3 / JavaScript ES2022
├── Three.js 3.145+ (3D rendering)
├── WebXR API (AR)
├── Service Worker (PWA caching)
└── MediaPipe (image tracking) ⭐ Alternativa: three-ar
```

### Hosting
```
Local Network
├── Notebook (Node.js server)
├── Router WiFi 5Ghz/2.4Ghz
└── Móviles (Cliente AR)
```

---

## 2. ESTRUCTURA DE CARPETAS

```
tambo-ar/
├── backend/
│   ├── server.js                 # Punto de entrada
│   ├── package.json
│   ├── vite.config.js
│   ├── .env.example
│   │
│   ├── routes/
│   │   ├── api.js               # Rutas API (/api/*)
│   │   ├── connect.js           # Pantalla conexión
│   │   ├── ar.js                # Pantalla AR
│   │   └── assets.js            # Streaming assets
│   │
│   ├── middleware/
│   │   ├── auth.js              # Validar sessionId
│   │   ├── cors.js
│   │   └── rateLimit.js
│   │
│   ├── utils/
│   │   ├── session.js           # Session manager
│   │   ├── qrGenerator.js       # QR dinámico
│   │   └── logger.js            # Logging
│   │
│   └── public/
│       ├── connect.html          # Pantalla conexión
│       ├── ar.html              # Pantalla AR
│       │
│       ├── js/
│       │   ├── ar-core.js       # Lógica AR principal
│       │   ├── tracking.js      # Image tracking
│       │   ├── animations.js    # Control animaciones
│       │   ├── ui.js            # UI controls
│       │   └── service-worker.js
│       │
│       ├── css/
│       │   ├── common.css
│       │   ├── ar.css
│       │   └── responsive.css
│       │
│       ├── models/
│       │   └── tambo-camar.glb  # Modelo comprimido
│       │
│       ├── images/
│       │   ├── maqueta-marker.jpg  # Para tracking
│       │   ├── logo.png
│       │   └── markers/
│       │
│       ├── libs/
│       │   ├── three.min.js
│       │   ├── three-examples/ (loaders, etc)
│       │   ├── mediapipe/
│       │   └── qrcode.min.js
│       │
│       └── manifest.json        # PWA manifest

├── docs/
│   ├── SETUP.md                 # Guía de instalación
│   ├── CALIBRATION.md           # Calibración maqueta
│   ├── TROUBLESHOOTING.md
│   └── API.md

└── .github/
    └── workflows/
        └── ci.yml              # Testing automático
```

---

## 3. FLUJO DE COMUNICACIÓN

```
┌────────────────┐
│   DISPOSITIVO  │
│    MÓVIL       │
└────────┬───────┘
         │
    [1] Escanea QR1
         │
         ↓
    GET /connect
         │
         ↓
┌────────────────────────────────────────┐
│  Pantalla: Conexión exitosa            │
│  SessionID: abc123xyz                  │
│  IP Server: 192.168.1.100:8080         │
│  [QR2 dinámico generado]               │
└────────┬───────────────────────────────┘
         │
    [2] Escanea QR2 (o manual)
         │
         ↓
    GET /ar?session=abc123xyz
         │
         ↓
┌────────────────────────────────────────┐
│  Página AR cargada                     │
│  ├─ Solicita permiso cámara            │
│  ├─ Carga modelo GLB (streaming)       │
│  ├─ Carga imagen de tracking           │
│  ├─ Inicia WebXR                       │
│  └─ Inicia detección de maqueta        │
└────────┬───────────────────────────────┘
         │
    [3] Usuario ubica maqueta en pantalla
         │
         ↓
┌────────────────────────────────────────┐
│  AR ACTIVO                             │
│  ├─ Modelo anclado sobre maqueta       │
│  ├─ Animaciones reproduciendo          │
│  └─ Gestos de interacción activos      │
└────────────────────────────────────────┘
```

---

## 4. BACKEND - IMPLEMENTACIÓN

### 4.1 server.js (Punto de entrada)

```javascript
import express from 'express';
import compression from 'compression';
import cors from 'cors';
import { fileURLToPath } from 'url';
import path from 'path';
import fs from 'fs';
import { v4 as uuidv4 } from 'uuid';
import QRCode from 'qrcode';

const app = express();
const __dirname = path.dirname(fileURLToPath(import.meta.url));

// ── MIDDLEWARE ──
app.use(compression());
app.use(cors());
app.use(express.json());
app.use(express.static(path.join(__dirname, 'public')));

// ── SESSION STORE (En-memory, considera Redis en produccción) ──
const sessions = new Map();

const SESSION_DURATION = 30 * 60 * 1000; // 30 minutos

function createSession() {
  const sessionId = uuidv4().slice(0, 8);
  const session = {
    id: sessionId,
    createdAt: Date.now(),
    expiresAt: Date.now() + SESSION_DURATION,
    ip: null,
    arStarted: false
  };
  sessions.set(sessionId, session);

  // Limpiar sesiones expiradas
  setTimeout(() => sessions.delete(sessionId), SESSION_DURATION);

  return session;
}

// ── RUTAS ──

// GET /connect - Pantalla de conexión
app.get('/connect', (req, res) => {
  const session = createSession();
  const baseUrl = `http://${req.hostname}:${process.env.PORT || 8080}`;
  const arUrl = `${baseUrl}/ar?session=${session.id}`;

  res.render('connect', {
    sessionId: session.id,
    baseUrl,
    arUrl,
    qrDataUrl: undefined // Se generará en cliente si es necesario
  });
});

// GET /api/sessions/:sessionId - Obtener datos de sesión
app.get('/api/sessions/:sessionId', (req, res) => {
  const session = sessions.get(req.params.sessionId);

  if (!session) {
    return res.status(404).json({ error: 'Sesión no encontrada' });
  }

  if (session.expiresAt < Date.now()) {
    sessions.delete(req.params.sessionId);
    return res.status(401).json({ error: 'Sesión expirada' });
  }

  res.json({
    sessionId: session.id,
    expiresIn: session.expiresAt - Date.now(),
    arStarted: session.arStarted
  });
});

// POST /api/qr/ar - Generar QR para URL AR
app.post('/api/qr/ar', express.json(), async (req, res) => {
  const { sessionId } = req.body;
  const session = sessions.get(sessionId);

  if (!session) {
    return res.status(404).json({ error: 'Sesión no válida' });
  }

  try {
    const baseUrl = `http://${req.hostname}:${process.env.PORT || 8080}`;
    const arUrl = `${baseUrl}/ar?session=${sessionId}`;

    const qrDataUrl = await QRCode.toDataURL(arUrl, {
      errorCorrectionLevel: 'H',
      type: 'image/png',
      quality: 0.95,
      margin: 2,
      width: 300
    });

    res.json({ success: true, qrDataUrl, arUrl });
  } catch (err) {
    res.status(500).json({ error: 'Error generando QR', details: err.message });
  }
});

// GET /ar - Página AR
app.get('/ar', (req, res) => {
  const { session: sessionId } = req.query;
  const session = sessions.get(sessionId);

  if (!session) {
    return res.status(404).send('Sesión no válida. Escanea QR1 primero.');
  }

  session.arStarted = true;
  res.sendFile(path.join(__dirname, 'public', 'ar.html'));
});

// GET /models/tambo-camar.glb - Streaming del modelo con caché
app.get('/models/tambo-camar.glb', (req, res) => {
  const modelPath = path.join(__dirname, 'public', 'models', 'tambo-camar.glb');
  const stat = fs.statSync(modelPath);

  res.setHeader('Content-Type', 'model/gltf-binary');
  res.setHeader('Content-Length', stat.size);
  res.setHeader('Cache-Control', 'public, max-age=86400'); // 24h cache
  res.setHeader('ETag', `"${stat.ino}-${stat.mtime.getTime()}"`);

  // Soporte para Range requests (streaming)
  if (req.headers.range) {
    const range = req.headers.range;
    const parts = range.replace(/bytes=/, '').split('-');
    const start = parseInt(parts[0], 10);
    const end = parts[1] ? parseInt(parts[1], 10) : stat.size - 1;

    res.writeHead(206, {
      'Content-Range': `bytes ${start}-${end}/${stat.size}`,
      'Content-Length': end - start + 1
    });
    fs.createReadStream(modelPath, { start, end }).pipe(res);
  } else {
    fs.createReadStream(modelPath).pipe(res);
  }
});

// GET /config/scene.json - Config de escena AR
app.get('/config/scene.json', (req, res) => {
  const config = {
    model: {
      url: '/models/tambo-camar.glb',
      scale: 1.0,
      position: { x: 0, y: 0, z: 0 },
      rotation: { x: 0, y: 0, z: 0 }
    },
    tracking: {
      imageUrl: '/images/maqueta-marker.jpg',
      minConfidence: 0.7,
      maxConfidence: 1.0
    },
    ar: {
      referenceSpace: 'local',
      requiredFeatures: ['hit-test', 'dom-overlay'],
      domOverlay: { root: '#ui-overlay' }
    },
    performance: {
      targetFPS: 60,
      maxPixelRatio: 1.5,
      enableShadows: false,
      enablePostProcessing: false
    }
  };

  res.json(config);
});

// ── SERVER STARTUP ──
const PORT = process.env.PORT || 8080;
app.listen(PORT, () => {
  console.log(`
╔════════════════════════════════════════╗
║  🎯 Tambo de Camar AR Server Active    ║
╠════════════════════════════════════════╣
║  Local:   http://localhost:${PORT}        ║
║  Network: http://<LOCAL-IP>:${PORT}      ║
║  Status:  ✅ Ready                      ║
╚════════════════════════════════════════╝
  `);
  console.log('Acceder a /connect para iniciar');
});

// Graceful shutdown
process.on('SIGTERM', () => {
  console.log('SIGTERM recibido, cerrando...');
  process.exit(0);
});
```

### 4.2 package.json

```json
{
  "name": "tambo-ar-server",
  "version": "1.0.0",
  "type": "module",
  "scripts": {
    "dev": "node --watch server.js",
    "build": "vite build",
    "start": "node server.js",
    "preview": "vite preview"
  },
  "dependencies": {
    "express": "^4.18.2",
    "compression": "^1.7.4",
    "cors": "^2.8.5",
    "uuid": "^9.0.0",
    "qrcode": "^1.5.3"
  },
  "devDependencies": {
    "vite": "^5.0.0"
  }
}
```

---

## 5. FRONTEND - IMPLEMENTACIÓN AR

### 5.1 ar.html (Estructura base)

```html
<!DOCTYPE html>
<html lang="es">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0, viewport-fit=cover">
  <title>AR · Tambo de Camar</title>

  <link rel="manifest" href="/manifest.json">
  <link rel="stylesheet" href="/css/ar.css">
  <link rel="stylesheet" href="/css/responsive.css">
</head>
<body>
  <!-- Canvas para rendering 3D -->
  <canvas id="ar-canvas"></canvas>

  <!-- UI Overlay -->
  <div id="ui-overlay">
    <!-- Loading screen -->
    <div id="loading-screen" class="overlay">
      <div class="loader-container">
        <div class="spinner"></div>
        <p id="loading-text">Cargando experiencia AR...</p>
        <div class="progress-bar">
          <div id="progress-fill" class="progress"></div>
        </div>
      </div>
    </div>

    <!-- Instrucciones iniciales -->
    <div id="instructions-overlay" class="overlay">
      <div class="instructions-card">
        <h2>Coloca tu celular frente a la maqueta</h2>
        <p>Apunta la cámara hacia el Tambo de Camar impreso.</p>
        <p class="hint">El modelo se anclará automáticamente cuando lo detectemos.</p>
        <button id="close-instructions" class="btn-secondary">Entendido</button>
      </div>
    </div>

    <!-- Controles de animación -->
    <div id="animation-controls" class="controls-panel">
      <div class="control-group">
        <label for="animation-select">Animación:</label>
        <select id="animation-select">
          <option value="">-- Sin animar --</option>
        </select>
      </div>

      <div class="control-group">
        <button id="play-btn" class="btn-control" title="Play">▶</button>
        <button id="pause-btn" class="btn-control" title="Pause">⏸</button>
        <button id="stop-btn" class="btn-control" title="Stop">⏹</button>
      </div>

      <div class="control-group">
        <label for="speed-control">Velocidad:</label>
        <input type="range" id="speed-control" min="0.5" max="2" step="0.1" value="1">
      </div>
    </div>

    <!-- Status bar -->
    <div id="status-bar" class="status-bar">
      <span id="tracking-status" class="status-indicator">🔴 Detectando...</span>
      <span id="fps-counter" class="fps-counter">60 FPS</span>
    </div>

    <!-- Toast notifications -->
    <div id="toast" class="toast"></div>

    <!-- Debug panel (oculto por defecto) -->
    <div id="debug-panel" class="debug-panel">
      <h3>Debug</h3>
      <pre id="debug-log"></pre>
      <button id="close-debug" class="btn-small">Cerrar</button>
    </div>
  </div>

  <!-- Scripts -->
  <script src="/libs/three.min.js"></script>
  <script src="/libs/mediapipe/solutions/drawing_utils.js"></script>
  <script src="/libs/mediapipe/solutions/objectron.js"></script>

  <script type="module" src="/js/ar-core.js"></script>
</body>
</html>
```

### 5.2 ar-core.js (Lógica AR principal)

```javascript
import * as THREE from 'https://cdn.jsdelivr.net/npm/three@r145/build/three.module.js';
import { GLTFLoader } from 'https://cdn.jsdelivr.net/npm/three@r145/examples/jsm/loaders/GLTFLoader.js';
import { DRACOLoader } from 'https://cdn.jsdelivr.net/npm/three@r145/examples/jsm/loaders/DRACOLoader.js';

// ── CONSTANTES ──
const CANVAS_ID = 'ar-canvas';
const SESSION_DURATION = 30 * 60 * 1000;
const MODEL_URL = '/models/tambo-camar.glb';
const CONFIG_URL = '/config/scene.json';

// ── STATE ──
let session = null;
let model = null;
let mixer = null;
let scene = null;
let camera = null;
let renderer = null;
let animationActions = [];
let currentAnimationIndex = -1;

// ── INITIALIZATION ──

async function init() {
  try {
    // 1. Validar sesión
    const sessionId = new URLSearchParams(location.search).get('session');
    if (!sessionId) {
      redirectToConnect();
      return;
    }

    session = await validateSession(sessionId);
    if (!session) {
      redirectToConnect();
      return;
    }

    // 2. Cargar configuración
    const config = await fetch(CONFIG_URL).then(r => r.json());

    // 3. Setup Three.js
    setupThreeJS();

    // 4. Cargar modelo GLB
    await loadModel(config.model.url);

    // 5. Inicializar WebXR
    await initWebXR();

    // 6. Inicializar tracking de imagen
    // await initImageTracking(config.tracking.imageUrl);

    // 7. Setup UI
    setupUI();

    // 8. Iniciar render loop
    startRenderLoop(config);

    hideLoadingScreen();
    showInstructions();

  } catch (error) {
    console.error('[AR] Error inicialización:', error);
    showToast('Error: ' + error.message);
  }
}

// ── THREE.JS SETUP ──

function setupThreeJS() {
  const canvas = document.getElementById(CANVAS_ID);

  // Scene
  scene = new THREE.Scene();
  scene.background = null;

  // Camera
  camera = new THREE.PerspectiveCamera(
    75,
    window.innerWidth / window.innerHeight,
    0.01,
    1000
  );

  // Renderer
  renderer = new THREE.WebGLRenderer({
    canvas,
    antialias: true,
    alpha: true
  });
  renderer.setSize(window.innerWidth, window.innerHeight);
  renderer.setPixelRatio(Math.min(window.devicePixelRatio, 1.5)); // Limitar sobremuestreo
  renderer.xr.enabled = true;

  // Lighting
  const ambientLight = new THREE.AmbientLight(0xffffff, 1.2);
  scene.add(ambientLight);

  const directionalLight = new THREE.DirectionalLight(0xffffff, 0.8);
  directionalLight.position.set(5, 10, 5);
  scene.add(directionalLight);

  // Window resize
  window.addEventListener('resize', onWindowResize);
}

// ── CARGAR MODELO GLB ──

async function loadModel(modelUrl) {
  return new Promise((resolve, reject) => {
    const loader = new GLTFLoader();

    // Opcional: Draco decoder para compresión
    const dracoLoader = new DRACOLoader();
    dracoLoader.setDecoderPath('https://www.gstatic.com/draco/versioned/decoders/1.5.5/');
    loader.setDRACOLoader(dracoLoader);

    // Progress tracking
    loader.load(
      modelUrl,
      (gltf) => {
        model = gltf.scene;
        model.scale.set(1, 1, 1);
        model.position.set(0, 0, 0);

        scene.add(model);

        // Setup animaciones
        if (gltf.animations.length > 0) {
          mixer = new THREE.AnimationMixer(model);
          animationActions = gltf.animations.map(clip =>
            mixer.clipAction(clip)
          );
          populateAnimationSelect(gltf.animations);
        }

        console.log('[AR] Modelo cargado', {
          animations: gltf.animations.length
        });
        resolve();
      },
      (progress) => {
        const percent = Math.round((progress.loaded / progress.total) * 100);
        updateLoadingProgress(percent);
      },
      (error) => {
        console.error('[AR] Error cargando modelo:', error);
        reject(error);
      }
    );
  });
}

// ── WEBXR INITIALIZATION ──

async function initWebXR() {
  if (!navigator.xr) {
    throw new Error('WebXR no disponible en este dispositivo');
  }

  const supported = await navigator.xr.isSessionSupported('immersive-ar');
  if (!supported) {
    throw new Error('AR (immersive-ar) no soportado');
  }

  try {
    const xrSession = await navigator.xr.requestSession('immersive-ar', {
      requiredFeatures: ['hit-test'],
      optionalFeatures: ['dom-overlay'],
      domOverlay: { root: document.body }
    });

    renderer.xr.setSession(xrSession);

    console.log('[AR] WebXR sesión iniciada');
  } catch (error) {
    console.warn('[AR] WebXR no disponible, usando fallback 3D', error);
    // Fallback: renderizar en 3D normal sin AR
  }
}

// ── ANIMATION LOOP ──

function startRenderLoop(config) {
  const clock = new THREE.Clock();
  let frameCount = 0;
  let fpsTime = 0;

  function animate() {
    requestAnimationFrame(animate);

    const delta = clock.getDelta();

    // Update animations
    if (mixer) {
      mixer.update(delta);
    }

    // Update tracking (si está implementado)
    // updateTracking();

    // FPS counter
    frameCount++;
    fpsTime += delta;
    if (fpsTime >= 1) {
      document.getElementById('fps-counter').textContent = frameCount + ' FPS';
      frameCount = 0;
      fpsTime = 0;
    }

    // Render
    renderer.render(scene, camera);
  }

  animate();
}

// ── UI SETUP ──

function setupUI() {
  const playBtn = document.getElementById('play-btn');
  const pauseBtn = document.getElementById('pause-btn');
  const stopBtn = document.getElementById('stop-btn');
  const animSelect = document.getElementById('animation-select');
  const speedControl = document.getElementById('speed-control');

  playBtn?.addEventListener('click', playCurrentAnimation);
  pauseBtn?.addEventListener('click', pauseAnimation);
  stopBtn?.addEventListener('click', stopAnimation);
  animSelect?.addEventListener('change', selectAnimation);
  speedControl?.addEventListener('input', changeAnimationSpeed);

  document.getElementById('close-instructions')?.addEventListener('click', () => {
    document.getElementById('instructions-overlay').classList.add('hidden');
  });
}

function populateAnimationSelect(animations) {
  const select = document.getElementById('animation-select');
  animations.forEach((clip, idx) => {
    const option = document.createElement('option');
    option.value = idx;
    option.textContent = clip.name || `Animación ${idx + 1}`;
    select.appendChild(option);
  });
}

function selectAnimation(event) {
  currentAnimationIndex = parseInt(event.target.value);
}

function playCurrentAnimation() {
  if (currentAnimationIndex >= 0 && animationActions[currentAnimationIndex]) {
    animationActions[currentAnimationIndex].reset();
    animationActions[currentAnimationIndex].play();
  }
}

function pauseAnimation() {
  if (currentAnimationIndex >= 0 && animationActions[currentAnimationIndex]) {
    animationActions[currentAnimationIndex].paused = true;
  }
}

function stopAnimation() {
  if (currentAnimationIndex >= 0 && animationActions[currentAnimationIndex]) {
    animationActions[currentAnimationIndex].stop();
  }
}

function changeAnimationSpeed(event) {
  const speed = parseFloat(event.target.value);
  if (mixer) {
    mixer.timeScale = speed;
  }
}

// ── UTILIDADES ──

async function validateSession(sessionId) {
  try {
    const response = await fetch(`/api/sessions/${sessionId}`);
    if (response.ok) {
      return await response.json();
    }
  } catch (error) {
    console.error('[AR] Error validando sesión:', error);
  }
  return null;
}

function redirectToConnect() {
  window.location.href = '/connect';
}

function updateLoadingProgress(percent) {
  const fill = document.getElementById('progress-fill');
  if (fill) fill.style.width = percent + '%';
}

function hideLoadingScreen() {
  const screen = document.getElementById('loading-screen');
  if (screen) screen.classList.add('hidden');
}

function showInstructions() {
  const instructions = document.getElementById('instructions-overlay');
  if (instructions) instructions.classList.remove('hidden');
}

function showToast(message) {
  const toast = document.getElementById('toast');
  if (toast) {
    toast.textContent = message;
    toast.classList.add('visible');
    setTimeout(() => toast.classList.remove('visible'), 3000);
  }
}

function onWindowResize() {
  camera.aspect = window.innerWidth / window.innerHeight;
  camera.updateProjectionMatrix();
  renderer.setSize(window.innerWidth, window.innerHeight);
}

function updateTracking(isTracking) {
  const status = document.getElementById('tracking-status');
  if (status) {
    status.textContent = isTracking ? '🟢 Detectado' : '🔴 Detectando...';
  }
}

// ── START ──

window.addEventListener('load', init);

// PWA registration
if ('serviceWorker' in navigator) {
  navigator.serviceWorker.register('/js/service-worker.js')
    .then(reg => console.log('[PWA] Service Worker registrado'))
    .catch(err => console.log('[PWA] Error:', err));
}
```

### 5.3 ar.css (Estilos)

```css
* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

html, body {
  width: 100%;
  height: 100%;
  overflow: hidden;
}

#ar-canvas {
  display: block;
  width: 100%;
  height: 100%;
}

/* ── UI OVERLAY ── */
#ui-overlay {
  position: fixed;
  inset: 0;
  z-index: 100;
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif;
  color: white;
}

.overlay {
  position: fixed;
  inset: 0;
  background: rgba(0, 0, 0, 0.7);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 110;
  transition: opacity 0.3s;
}

.overlay.hidden {
  opacity: 0;
  pointer-events: none;
}

.loader-container {
  text-align: center;
}

.spinner {
  width: 50px;
  height: 50px;
  border: 3px solid rgba(255, 255, 255, 0.3);
  border-radius: 50%;
  border-top: 3px solid white;
  animation: spin 1s linear infinite;
  margin-bottom: 20px;
}

@keyframes spin {
  to { transform: rotate(360deg); }
}

.progress-bar {
  width: 200px;
  height: 4px;
  background: rgba(255, 255, 255, 0.2);
  border-radius: 2px;
  overflow: hidden;
  margin-top: 15px;
}

.progress {
  height: 100%;
  background: linear-gradient(90deg, #4dd0e1, #00bcd4);
  width: 0%;
  transition: width 0.3s;
}

/* ── INSTRUCTIONS ── */
.instructions-card {
  background: rgba(0, 0, 0, 0.9);
  padding: 30px;
  border-radius: 16px;
  max-width: 90%;
  text-align: center;
  border: 1px solid rgba(255, 255, 255, 0.1);
}

.instructions-card h2 {
  font-size: 18px;
  margin-bottom: 15px;
}

.instructions-card p {
  font-size: 14px;
  opacity: 0.9;
  margin-bottom: 10px;
}

.instructions-card .hint {
  font-size: 12px;
  opacity: 0.7;
  font-style: italic;
}

/* ── CONTROLES ── */
.controls-panel {
  position: fixed;
  bottom: 20px;
  left: 20px;
  right: 20px;
  background: rgba(0, 0, 0, 0.85);
  border: 1px solid rgba(255, 255, 255, 0.1);
  border-radius: 12px;
  padding: 15px;
  backdrop-filter: blur(10px);
  max-width: 400px;
}

.control-group {
  display: flex;
  align-items: center;
  gap: 10px;
  margin-bottom: 10px;
  font-size: 13px;
}

.control-group:last-child {
  margin-bottom: 0;
}

.control-group label {
  flex: 0 0 auto;
  min-width: 70px;
}

.control-group select,
.control-group input[type="range"] {
  flex: 1;
  padding: 6px;
  background: rgba(255, 255, 255, 0.1);
  border: 1px solid rgba(255, 255, 255, 0.2);
  color: white;
  border-radius: 6px;
  font-size: 12px;
}

.btn-control {
  padding: 8px 12px;
  background: rgba(77, 208, 225, 0.8);
  color: black;
  border: none;
  border-radius: 6px;
  cursor: pointer;
  font-weight: bold;
  transition: background 0.2s;
}

.btn-control:active {
  background: rgba(77, 208, 225, 1);
  transform: scale(0.95);
}

/* ── STATUS BAR ── */
.status-bar {
  position: fixed;
  top: 20px;
  left: 20px;
  right: 20px;
  display: flex;
  justify-content: space-between;
  align-items: center;
  font-size: 12px;
  padding: 10px;
  background: rgba(0, 0, 0, 0.7);
  border-radius: 8px;
  backdrop-filter: blur(10px);
}

.status-indicator {
  padding: 4px 8px;
  background: rgba(255, 255, 255, 0.1);
  border-radius: 4px;
}

.fps-counter {
  font-family: 'Courier New', monospace;
  opacity: 0.7;
}

/* ── TOAST ── */
.toast {
  position: fixed;
  bottom: 30px;
  left: 50%;
  transform: translateX(-50%) translateY(100px);
  background: rgba(0, 0, 0, 0.9);
  color: white;
  padding: 12px 20px;
  border-radius: 8px;
  font-size: 13px;
  opacity: 0;
  pointer-events: none;
  transition: all 0.3s;
  z-index: 200;
}

.toast.visible {
  opacity: 1;
  transform: translateX(-50%) translateY(0);
  pointer-events: auto;
}

/* ── DEBUG PANEL ── */
.debug-panel {
  position: fixed;
  bottom: 20px;
  right: 20px;
  background: rgba(0, 0, 0, 0.95);
  color: #0f0;
  padding: 15px;
  border-radius: 8px;
  font-family: monospace;
  font-size: 11px;
  max-width: 300px;
  max-height: 300px;
  overflow-y: auto;
  display: none;
}

.debug-panel.visible {
  display: block;
}

#debug-log {
  margin: 10px 0;
  line-height: 1.4;
}

/* ── RESPONSIVE ── */
@media (max-width: 600px) {
  .controls-panel {
    left: 10px;
    right: 10px;
    max-width: none;
  }

  .status-bar {
    left: 10px;
    right: 10px;
  }
}
```

---

## 6. IMAGE TRACKING

**Opción recomendada: MediaPipe**

```javascript
// tracking.js
import { Objectron } from 'https://cdn.jsdelivr.net/npm/@mediapipe/objectron@0.4.1667953766471/objectron.js';
import { drawConnectors, drawLandmarks } from 'https://cdn.jsdelivr.net/npm/@mediapipe/drawing_utils@0.4/drawing_utils.js';

const objectron = new Objectron({
  locateFile: (file) => {
    return `https://cdn.jsdelivr.net/npm/@mediapipe/objectron@latest/${file}`;
  }
});

objectron.onResults(onObjectronResults);

async function initObjectron() {
  await objectron.initialize();
  detectObjects();
}

function onObjectronResults(results) {
  if (results.objectDetections && results.objectDetections.length > 0) {
    const detection = results.objectDetections[0];
    console.log('Objeto detectado:', detection);
    updateModelPose(detection);
  }
}

function detectObjects() {
  const canvas = document.getElementById('ar-canvas');
  objectron.send({ image: canvas });
  requestAnimationFrame(detectObjects);
}
```

---

## 7. DEPLOYMENT EN SERVIDOR LOCAL

### Script de inicio (start.sh)

```bash
#!/bin/bash

# Obtener IP local
IP=$(hostname -I | awk '{print $1}')
PORT=8080

echo "╔═══════════════════════════════════════╗"
echo "║ 🎯 Tambo de Camar AR - Server         ║"
echo "╠═══════════════════════════════════════╣"
echo "║ Local:   http://localhost:${PORT}     ║"
echo "║ Network: http://${IP}:${PORT}        ║"
echo "║ QR Link: /connect                    ║"
echo "╚═══════════════════════════════════════╝"
echo ""

# Instalar dependencias si no existen
if [ ! -d "node_modules" ]; then
  echo "📦 Instalando dependencias..."
  npm install
fi

# Iniciar servidor
npm run start
```

---

## 8. CHECKLIST PRE-PRODUCCIÓN

```
[ ] Modelo GLB optimizado y comprimido
[ ] Imagen de maqueta en alta resolución
[ ] WebXR funcionando en iOS 17.4+
[ ] WebXR funcionando en Android Chrome 88+
[ ] Image tracking <100ms de latencia
[ ] Animaciones reproduciendo fluídamente
[ ] Performance >30 FPS en gama media
[ ] Responsivo en portrait/landscape
[ ] PWA funcional (offline capable)
[ ] Sesiones expiran correctamente
[ ] QR1 y QR2 generados y validados
[ ] Testing en 5+ dispositivos reales
[ ] Documentación completa
[ ] Logs y debug panel funcionales
```

---

## 9. REFERENCIAS Y RECURSOS

- [WebXR API](https://www.w3.org/TR/webxr/)
- [Three.js AR Examples](https://threejs.org/examples/#webxr_ar_cones)
- [MediaPipe Solutions](https://solutions.mediapipe.dev/web)
- [WebGL Performance](https://www.khronos.org/webgl/wiki/Optimization-Tips)

