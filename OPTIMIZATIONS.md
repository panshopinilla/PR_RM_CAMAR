# ⚡ GUÍA DE OPTIMIZACIONES - Tambo de Camar AR

## 1. OPTIMIZACIONES DEL MODELO 3D

### 1.1 Compresión GLB (Ya hecha en app anterior)
```
✅ DRACO compression (reduce 80-90%)
✅ Quantización de vertices
✅ Compresión de texturas WebP
✅ Eliminación de datos innecesarios
```

**Verificar tamaño final:**
```bash
ls -lh public/models/tambo-camar.glb
# Objetivo: < 20MB para carga rápida en WiFi local
```

### 1.2 Level of Detail (LOD)
```javascript
// Si el modelo es complejo, implementar LOD
const lod = new THREE.LOD();

// Versión detallada (0-50m)
lod.addLevel(detailedModel, 0);

// Versión media (50-150m)
lod.addLevel(mediumModel, 50);

// Versión simplificada (150m+)
lod.addLevel(simpleModel, 150);

scene.add(lod);
```

### 1.3 Culling de mallas invisibles
```javascript
// Desabilitar mallas no visibles desde la cámara AR
model.traverse((mesh) => {
  if (mesh.isMesh) {
    mesh.frustumCulled = true;
  }
});
```

---

## 2. OPTIMIZACIONES DE RENDERIZADO

### 2.1 Configuración del renderer
```javascript
// ar-core.js - setupThreeJS()

// Limitar pixel ratio (evita sobremuestreo)
renderer.setPixelRatio(Math.min(window.devicePixelRatio, 1.5));

// Shadows = MUY CAROS en AR, desabilitar
renderer.shadowMap.enabled = false;

// Antialias suave pero eficiente
renderer.antialias = true;

// Tone mapping simple
renderer.toneMapping = THREE.NoToneMapping;

// Clear alpha para WebXR (transparencia)
renderer.setClearAlpha(0);
```

### 2.2 Lighting optimizado
```javascript
// Usar solo iluminación ambiental + 1 directional light

const ambientLight = new THREE.AmbientLight(0xffffff, 1.0);
scene.add(ambientLight);

const dirLight = new THREE.DirectionalLight(0xffffff, 0.6);
dirLight.position.set(5, 10, 5);
// NO usar shadows (demasiado caro)
dirLight.castShadow = false;
scene.add(dirLight);
```

### 2.3 Texture optimization
```javascript
// En GLTFLoader progress
loader.load(modelUrl, (gltf) => {
  gltf.scene.traverse((mesh) => {
    if (mesh.isMesh) {
      // Anisotropic filtering (mejor calidad + pequeño costo)
      if (mesh.material.map) {
        mesh.material.map.anisotropy = 4;
      }

      // Disable envMap si no es necesario
      mesh.material.envMapIntensity = 0.5;
    }
  });
});
```

### 2.4 Garbage collection
```javascript
// Limpiar recursos cuando no se usan
function dispose(object) {
  if (object.geometry) object.geometry.dispose();
  if (object.material) {
    if (Array.isArray(object.material)) {
      object.material.forEach(m => m.dispose());
    } else {
      object.material.dispose();
    }
  }
  object.traverse(child => {
    if (child.geometry) child.geometry.dispose();
  });
}
```

---

## 3. OPTIMIZACIONES DE NETWORK

### 3.1 Caching agresivo
```javascript
// server.js - Ruta de modelo
app.get('/models/tambo-camar.glb', (req, res) => {
  const modelPath = path.join(__dirname, 'public', 'models', 'tambo-camar.glb');
  const stat = fs.statSync(modelPath);

  res.setHeader('Cache-Control', 'public, max-age=86400'); // 24h
  res.setHeader('ETag', `"${stat.ino}-${stat.mtime.getTime()}"`);
  res.setHeader('Last-Modified', stat.mtime.toUTCString());

  // Soportar Range requests (resume de descargas)
  if (req.headers.range) {
    // ... implementar range request
  }

  fs.createReadStream(modelPath).pipe(res);
});
```

### 3.2 Compresión HTTP
```javascript
// server.js - En middleware
import compression from 'compression';

app.use(compression({
  level: 9,           // Máxima compresión
  threshold: 1024,    // Comprimir si >1KB
  filter: (req, res) => {
    // No comprimir GLB (ya está comprimido)
    if (req.url.includes('.glb')) {
      return false;
    }
    return compression.filter(req, res);
  }
}));
```

### 3.3 Lazy loading de assets
```javascript
// Cargar modelo solo cuando se necesita
async function loadModelWhenNeeded() {
  const observer = new IntersectionObserver((entries) => {
    if (entries[0].isIntersecting) {
      loadModel('/models/tambo-camar.glb');
      observer.disconnect();
    }
  });

  observer.observe(document.getElementById('ar-canvas'));
}
```

### 3.4 Streaming de modelo
```javascript
// En lugar de descargar todo, hacer streaming
app.get('/models/tambo-camar.glb', (req, res) => {
  const chunkSize = 1024 * 1024; // 1MB chunks

  const stream = fs.createReadStream(modelPath, {
    highWaterMark: chunkSize
  });

  res.setHeader('Content-Type', 'model/gltf-binary');
  stream.pipe(res);
});
```

---

## 4. OPTIMIZACIONES DE ANIMATION

### 4.1 Update loop eficiente
```javascript
// ar-core.js - startRenderLoop()
function startRenderLoop(config) {
  const clock = new THREE.Clock();
  let lastTime = 0;

  function animate(time) {
    requestAnimationFrame(animate);

    const delta = Math.min(clock.getDelta(), 0.016); // Max 16ms (60fps)

    // Solo actualizar si cambió
    if (mixer && animationActions.some(a => a.isRunning())) {
      mixer.update(delta);
    }

    // Throttle rendering a 30 FPS en dispositivos lentos
    if (time - lastTime > 33) { // ~30fps
      renderer.render(scene, camera);
      lastTime = time;
    }
  }

  animate();
}
```

### 4.2 Animation clipping
```javascript
// Parar animaciones que no se ven
model.traverse((mesh) => {
  if (mesh.isMesh) {
    // Frustum culling para meshes
    mesh.frustumCulled = true;
  }
});
```

### 4.3 AnimationMixer pooling (para múltiples modelos)
```javascript
// Si hay múltiples modelos, reutilizar mixer
const mixerPool = new Map();

function getOrCreateMixer(object) {
  if (!mixerPool.has(object)) {
    mixerPool.set(object, new THREE.AnimationMixer(object));
  }
  return mixerPool.get(object);
}
```

---

## 5. OPTIMIZACIONES DE AR/WEBXR

### 5.1 Image tracking - Pose smoothing
```javascript
// tracking.js
class PoseFilter {
  constructor(alpha = 0.8) {
    this.alpha = alpha;
    this.lastPose = null;
  }

  smooth(newPose) {
    if (!this.lastPose) {
      this.lastPose = { ...newPose };
      return newPose;
    }

    // LERP para posición
    const smoothed = {
      position: {
        x: this.lerp(this.lastPose.position.x, newPose.position.x, this.alpha),
        y: this.lerp(this.lastPose.position.y, newPose.position.y, this.alpha),
        z: this.lerp(this.lastPose.position.z, newPose.position.z, this.alpha)
      },
      quaternion: newPose.quaternion // Usar SLERP si es posible
    };

    this.lastPose = smoothed;
    return smoothed;
  }

  lerp(a, b, t) {
    return a + (b - a) * t;
  }
}
```

### 5.2 Reduce pose update frequency
```javascript
// En lugar de actualizar pose cada frame, hacerlo cada N frames
let poseUpdateCounter = 0;
const POSE_UPDATE_INTERVAL = 3; // Cada 3 frames

function updateTracking() {
  poseUpdateCounter++;
  if (poseUpdateCounter % POSE_UPDATE_INTERVAL === 0) {
    // Actualizar pose del modelo
    updateModelPose(currentPose);
  }
}
```

### 5.3 Optimize hit-test
```javascript
// WebXR hit testing es caro, reducir frecuencia
async function performHitTest(frame) {
  if (!hitTestSource) return;

  const hitTestResults = frame.getHitTestResults(hitTestSource);
  if (hitTestResults.length > 0) {
    const hit = hitTestResults[0];
    // Usar resultado
  }
}

// Llamar cada 2-3 frames, no cada frame
let hitTestCounter = 0;
if (hitTestCounter++ % 3 === 0) {
  await performHitTest(frame);
}
```

---

## 6. OPTIMIZACIONES MÓVIL-ESPECÍFICAS

### 6.1 iOS (Safari)
```javascript
// Detectar iOS
const isIOS = /iPad|iPhone|iPod/.test(navigator.userAgent);

if (isIOS) {
  // Limitar más agresivamente en iOS
  renderer.setPixelRatio(1.0); // No 1.5+

  // Reducir draw calls
  renderer.sortObjects = false;

  // Desabilitar post-processing
  enablePostProcessing = false;
}
```

### 6.2 Android
```javascript
// Detectar capacidad
const hasARCore = navigator.xr?.isSessionSupported('immersive-ar');

if (hasARCore) {
  // Usar full power si disponible
  renderer.setPixelRatio(1.5);
} else {
  // Fallback conservador
  renderer.setPixelRatio(1.0);
}
```

### 6.3 Gesture optimization
```javascript
// Debounce de gestos para no saturar
function debounce(fn, delay) {
  let timeout;
  return (...args) => {
    clearTimeout(timeout);
    timeout = setTimeout(() => fn(...args), delay);
  };
}

const onTouchMove = debounce((e) => {
  updateModelRotation(e);
}, 16); // Max 60Hz
```

### 6.4 Battery optimization
```javascript
// Reducir FPS si batería baja
if (navigator.getBattery) {
  navigator.getBattery().then((battery) => {
    if (battery.level < 0.2) {
      targetFPS = 30; // Reducir a 30 FPS
    }
  });
}
```

---

## 7. OPTIMIZACIONES PWA

### 7.1 Service Worker caching estrategy
```javascript
// service-worker.js
const CACHE_VERSION = 'tambo-ar-v1';
const ASSETS_TO_CACHE = [
  '/',
  '/ar.html',
  '/connect.html',
  '/css/ar.css',
  '/js/ar-core.js',
  '/models/tambo-camar.glb',
  '/images/maqueta-marker.jpg'
];

self.addEventListener('install', (event) => {
  event.waitUntil(
    caches.open(CACHE_VERSION).then((cache) => {
      // Cachear críticos
      return cache.addAll(ASSETS_TO_CACHE);
    })
  );
  self.skipWaiting();
});

self.addEventListener('fetch', (event) => {
  const { request } = event;

  // Para GLB: cache-first pero validar actualización
  if (request.url.includes('.glb')) {
    event.respondWith(
      caches.match(request).then((response) => {
        return response || fetch(request).then((response) => {
          caches.open(CACHE_VERSION).then((cache) => {
            cache.put(request, response.clone());
          });
          return response;
        });
      })
    );
    return;
  }

  // Para HTML/CSS/JS: network-first
  event.respondWith(
    fetch(request)
      .catch(() => caches.match(request))
  );
});
```

### 7.2 Lazy load Service Worker
```javascript
// Solo registrar SW cuando sea necesario
window.addEventListener('load', () => {
  if ('serviceWorker' in navigator) {
    // Registrar con delay para no bloquear
    setTimeout(() => {
      navigator.serviceWorker.register('/js/service-worker.js')
        .catch(err => console.log('[PWA] Error:', err));
    }, 2000);
  }
});
```

---

## 8. MONITOREO DE PERFORMANCE

### 8.1 FPS Monitor
```javascript
class FPSMonitor {
  constructor() {
    this.fps = 0;
    this.lastTime = performance.now();
    this.frames = 0;
  }

  update() {
    this.frames++;
    const now = performance.now();
    const delta = now - this.lastTime;

    if (delta >= 1000) {
      this.fps = this.frames;
      this.frames = 0;
      this.lastTime = now;
    }

    return this.fps;
  }

  getWarning() {
    if (this.fps < 30) return '⚠️ Low FPS';
    if (this.fps < 60) return '⚠️ FPS bajo';
    return '✅ OK';
  }
}

const fpsMonitor = new FPSMonitor();
```

### 8.2 Memory monitoring
```javascript
// Chrome only
if (performance.memory) {
  console.log('Memory:', {
    used: (performance.memory.usedJSHeapSize / 1048576).toFixed(2) + ' MB',
    limit: (performance.memory.jsHeapSizeLimit / 1048576).toFixed(2) + ' MB',
    percentage: ((performance.memory.usedJSHeapSize / performance.memory.jsHeapSizeLimit) * 100).toFixed(1) + '%'
  });
}
```

### 8.3 Network monitoring
```javascript
// Medir carga de assets
const observer = new PerformanceObserver((list) => {
  for (const entry of list.getEntries()) {
    console.log(`[Network] ${entry.name}:`, {
      duration: entry.duration.toFixed(2) + 'ms',
      size: (entry.transferSize / 1024).toFixed(2) + 'KB'
    });
  }
});

observer.observe({ entryTypes: ['resource'] });
```

---

## 9. BENCHMARKING

### Herramientas recomendadas
```
✓ Chrome DevTools (Network, Performance tabs)
✓ Lighthouse (PWA, Performance audit)
✓ WebPageTest (Remote device testing)
✓ Profiler integrado de Three.js
```

### Comandos útiles
```bash
# Analizar tamaño de bundle
npm run build && du -sh dist/

# Performance audit
lighthouse http://192.168.1.100:8080/ar --view

# Memory profiling
node --inspect server.js
# Luego abrir chrome://inspect en Chrome
```

---

## 10. TARGETS RECOMENDADOS

| Métrica | Target | Crit. |
|---------|--------|-------|
| **Carga inicial** | <2s | <3s |
| **TTI (Interactive)** | <3s | <5s |
| **FPS en AR** | 60 | 30+ |
| **Memory (mobile)** | <100MB | <150MB |
| **Model size** | <15MB | <30MB |
| **Cache hit rate** | >95% | >80% |
| **Tracking latency** | <100ms | <200ms |

---

## 11. CHECKLIST DE OPTIMIZACIÓN

```
MODELO 3D:
[✓] GLB comprimido con DRACO
[✓] Texturas optimizadas
[ ] LOD implementado (si necesario)
[ ] Animaciones sin overhead

RENDERIZADO:
[ ] Pixel ratio limitado a 1.5
[ ] Shadows desabilitados
[ ] Lighting mínima (ambient + 1 directional)
[ ] Culling de mallas habilitado

NETWORK:
[ ] Caching HTTP (max-age: 86400)
[ ] Compresión gzip
[ ] Range requests implementados
[ ] Streaming de GLB

AR/WEBXR:
[ ] Pose smoothing implementado
[ ] Update frequency optimizada
[ ] Hit-test frequency reducida
[ ] Gesture debouncing

PWA:
[ ] Service Worker con cacheo inteligente
[ ] Offline capability
[ ] Manifest.json optimizado

MOBILE:
[ ] iOS-specific optimizations
[ ] Android gama media soportado
[ ] Battery optimization
[ ] Gesture performance

TESTING:
[ ] FPS monitor integrado
[ ] Memory leaks checkeados
[ ] Network waterfall analizado
[ ] 5+ dispositivos testeados
```

