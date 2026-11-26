# Gestor de Tareas PWA (Offline-First)

Aplicación Web Progresiva (**PWA**) moderna desarrollada con un enfoque **Offline-First**, diseñada para funcionar de forma confiable sin conexión a internet y sincronizarse de manera automática e idempotente con el servidor al restaurarse la conectividad.

El proyecto incorpora persistencia local avanzada, criptografía en el navegador, integración con hardware del dispositivo y notificaciones push web con claves VAPID.

---

## 🚀 Características Principales

- **Offline-First & PWA:** Instalable como aplicación nativa en dispositivos móviles y de escritorio mediante `manifest.json`. Cacheo predictivo y dinámico gestionado por Service Worker.
- **Persistencia con IndexedDB v2:** Base de datos transaccional con control de versiones y migraciones de esquema (almacenes dedicados para `tareas` y cola `outbox`).
- **Notas Privadas con Web Crypto API:** Cifrado simétrico de notas confidenciales en el cliente utilizando **AES-GCM** de 256 bits y derivación de clave desde contraseña mediante **PBKDF2** (con sal aleatoria e iteraciones criptográficas). La clave vive exclusivamente en memoria de la sesión.
- **Hardware y Sensores del Dispositivo:**
  - **MediaDevices API:** Captura de fotografías en tiempo real desde la cámara (`getUserMedia`).
  - **Geolocation API:** Registro de coordenadas geográficas (latitud, longitud y precisión) en cada tarea.
  - **DeviceMotion & DeviceOrientation API:** Detección de sacudidas (*shake to undo*) para deshacer la última eliminación de tareas e inspección de inclinación (\(\beta\) y \(\gamma\)).
- **Sincronización en Segundo Plano (Background Sync API):** Cola de salida (*outbox*) persistente que almacena los cambios locales mientras no hay conexión y los transmite automáticamente al servidor con garantías de idempotencia (`idCambio`).
- **Notificaciones Push con VAPID:** Mensajería y recordatorios remotos mediante Push API y Web Push con soporte de claves VAPID públicas y privadas.

---

## 🏗 Arquitectura y Tecnologías

```
┌─────────────────────────────────────────────────────────────┐
│                       Interfaz Web (UI)                     │
│               index.html · app.js · css/estilos.css         │
└──────────────┬───────────────────────────────┬──────────────┘
               │                               │
               ▼                               ▼
┌──────────────────────────────┐ ┌─────────────────────────────┐
│  Service Worker (sw.js)      │ │   Almacenamiento Local      │
│  - Precaching (Workbox)      │ │   - IndexedDB v2 (tareas &  │
│  - Push & Notification click │ │     outbox de sincronización│
│  - Background Sync ("sync")  │ │   - Web Crypto (AES-GCM)    │
└──────────────┬───────────────┘ └─────────────┬───────────────┘
               │                               │
               ▼                               ▼
┌─────────────────────────────────────────────────────────────┐
│               Servidor Node.js (servidor/server.js)         │
│          - Entrega estática http://localhost:3000           │
│          - Gestión de suscripciones y envío Web-Push        │
│          - Endpoint de sincronización REST (/api/tareas)    │
└─────────────────────────────────────────────────────────────┘
```

1. **Service Worker e Inyección de Manifiesto con Workbox (`InjectManifest`):**  
   El archivo de origen [`sw-propio.js`](file:///c:/repositorios/Proyecto_Gestor_de_Tareas/sw-propio.js) contiene la lógica nativa para eventos de ciclo de vida (`install`, `activate`), enrutamiento/estrategias de caché (`StaleWhileRevalidate`, `NetworkFirst`, `CacheFirst`), recepción de eventos `push`, clics sobre notificaciones y la sincronización en segundo plano (`sync`). Mediante `workbox-cli injectManifest`, se compila el archivo final [`sw.js`](file:///c:/repositorios/Proyecto_Gestor_de_Tareas/sw.js) inyectando la lista de activos precacheados con sus respectivos hashes de integridad.

2. **IndexedDB v2 y Web Crypto API:**  
   [`db.js`](file:///c:/repositorios/Proyecto_Gestor_de_Tareas/db.js) gestiona la base `gestor-tareas-db` en versión 2, manejando la actualización desde esquemas anteriores para incorporar el almacén `outbox` con índice `por-fecha`. En [`crypto.js`](file:///c:/repositorios/Proyecto_Gestor_de_Tareas/crypto.js), el cifrado de datos sensibles nunca envía textos en claro al almacenamiento: se deriva una CryptoKey volátil y se utiliza un vector de inicialización (IV) único de 12 bytes por cada cifrado.

3. **APIs del Dispositivo:**  
   [`dispositivo.js`](file:///c:/repositorios/Proyecto_Gestor_de_Tareas/dispositivo.js) solicita permisos bajo demanda para la cámara y la geolocalización, deteniendo los *tracks* del flujo de video tan pronto como se completa o cancela la captura. [`sensores.js`](file:///c:/repositorios/Proyecto_Gestor_de_Tareas/sensores.js) aplica filtros pasa-altos/cálculo de magnitud de aceleración para evitar falsos positivos y desconecta los escuchadores para proteger la batería del dispositivo.

4. **Push API & Background Sync:**  
   [`notificaciones.js`](file:///c:/repositorios/Proyecto_Gestor_de_Tareas/notificaciones.js) convierte la clave pública VAPID en un `Uint8Array` compatible con `pushManager.subscribe`. [`sync.js`](file:///c:/repositorios/Proyecto_Gestor_de_Tareas/sync.js) registra etiquetas de sincronización con `SyncManager` o activa un plan B basado en el evento `online` del navegador para clientes sin soporte nativo de Background Sync.

---

## 📂 Estructura del Proyecto

```text
Proyecto_Gestor_de_Tareas/
├── .gitignore                     # Configuración de exclusiones de Git
├── README.md                      # Documentación del proyecto
├── index.html                     # Marcado semántico de la aplicación
├── app.js                         # Orquestador del DOM y flujo de datos
├── tareas.js                      # Lógica de creación y renderizado de tareas
├── db.js                          # Capa de acceso y migraciones IndexedDB
├── crypto.js                      # Cifrado AES-GCM y derivación PBKDF2
├── dispositivo.js                 # Integración de cámara y geolocalización
├── sensores.js                    # Sensores de acelerómetro y orientación
├── notificaciones.js              # Gestión de notificaciones locales y Push
├── sync.js                        # Bandeja de salida y Background Sync
├── sw-propio.js                   # Código fuente del Service Worker
├── sw.js                          # Service Worker generado por Workbox
├── workbox-config.js              # Configuración de InjectManifest
├── package-lock.json              # Dependencias de herramientas locales
├── css/
│   └── estilos.css                # Estilos visuales de la aplicación
├── icons/                         # Iconos PWA (192px, 512px, maskable)
├── lib/                           # Librerías de Workbox en local
├── referencia-typescript/         # Definiciones y tipado TypeScript de referencia
└── servidor/                      # Servidor backend de pruebas en Node.js
    ├── package.json
    ├── package-lock.json
    ├── server.js                  # Servidor HTTP nativo + web-push
    ├── claves-vapid.json          # Claves VAPID locales (ignorado en Git)
    └── datos.json                 # Base de datos JSON de prueba (ignorado en Git)
```

---

## ⚙️ Requisitos e Instalación

### Prerrequisitos
- **Node.js** (versión 18 o superior recomendada).
- **npm** (incluido con Node.js).

### 1. Instalación de dependencias del servidor
Navega a la carpeta del servidor e instala las dependencias (`web-push`):

```bash
cd servidor
npm install
```

### 2. Ejecución de la aplicación
Inicia el servidor local:

```bash
node server.js
```

Abre tu navegador en:
👉 **[http://localhost:3000](http://localhost:3000)**

> **Nota sobre claves VAPID:** En la primera ejecución se generará automáticamente el archivo `servidor/claves-vapid.json`. Este archivo contiene las llaves criptográficas del servidor y se encuentra protegido por `.gitignore`.

---

## 🛠 Comandos de Desarrollo y Workbox

Cada vez que se modifique [`sw-propio.js`](file:///c:/repositorios/Proyecto_Gestor_de_Tareas/sw-propio.js) o cualquiera de los archivos estáticos precacheados (como [`css/estilos.css`](file:///c:/repositorios/Proyecto_Gestor_de_Tareas/css/estilos.css), [`index.html`](file:///c:/repositorios/Proyecto_Gestor_de_Tareas/index.html) o módulos `.js`), debes regenerar el Service Worker ejecutando el siguiente comando desde la raíz del proyecto:

```bash
npx workbox-cli injectManifest workbox-config.js
```

---

## 🧪 Pruebas Recomendadas

1. **Funcionamiento Offline:**
   - Abre las DevTools del navegador (`F12`), ve a la pestaña **Network** y selecciona el perfil **Offline**.
   - Agrega una nueva tarea: observarás que se almacena localmente y el indicador de sincronización marcará `1 cambio(s) pendiente(s)`.
   - Vuelve a activar **Online**: la aplicación sincronizará automáticamente la tarea pendiente con el servidor.

2. **Notas Cifradas:**
   - Ingresa una nota privada en cualquier tarea y pulsa **Guardar**.
   - Introduce una contraseña de cifrado en el cuadro de diálogo.
   - Pulsa **Ver nota**: solo se descifrará si la contraseña coincide con la utilizada al cifrar.

3. **Cámara y Geolocalización:**
   - Pulsa **Foto** para activar el visor de cámara en vivo y tomar una captura.
   - Pulsa **Ubicación** para adjuntar las coordenadas actuales obtenidas por GPS / red.

4. **Notificaciones Push y Sensores:**
   - Pulsa **Activar avisos** para conceder permisos de notificación.
   - Pulsa **Suscribir a push** y luego **Enviar push de prueba** para verificar la recepción de mensajes enviados desde el servidor Node.js.
   - En dispositivos móviles o emuladores de sensores en DevTools, activa **Agitar para deshacer** para restaurar una tarea eliminada mediante movimiento.
