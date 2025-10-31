## Informe del Backend del Software de Cámaras de Seguridad

Este informe describe el estado actual del backend del proyecto, enfocado en la visualización en vivo de cámaras IP y DVR/NVR. Incluye detalles sobre los archivos adjuntos, los **endpoints disponibles**, las **funcionalidades implementadas** y la **comunicación entre componentes**. El backend está construido con **Flask** y soporta hasta **128 canales simultáneos**, con énfasis en eficiencia y seguridad. Se detalla cada archivo adjunto para proporcionar una visión completa y evitar errores en el trabajo subsiguiente.

## Detalle de Archivos Adjuntos

A continuación, se describe cada archivo adjunto en su estado actual, incluyendo su propósito, contenido clave y cómo contribuye al sistema.

- **`config.py`** (backend/config/config.py):  
  Este archivo maneja las configuraciones globales del proyecto, cargando valores desde `.env` con fallbacks por defecto. Incluye logging básico, conexión a **PostgreSQL** (DB_HOST, DB_PORT, DB_USER, DB_PASS, DB_NAME → DATABASE_URL), parámetros de red (SCAN_PORTS como lista de enteros, ONVIF_TIMEOUT=5, DEFAULT_NETWORK_RANGE), límites de streaming (MAX_STREAMS=128, STREAM_TIMEOUT=300, CACHE_TTL=300, LOW_LATENCY_HLS=True), CORS (ALLOWED_ORIGINS como lista), paths absolutos (BASE_DIR, HLS_OUTPUT_DIR con makedirs), binario de FFmpeg (FFMPEG_BINARY), argumentos por defecto de FFmpeg (FFMPEG_DEFAULT_ARGS como lista optimizada para baja latencia y 128 canales), **nuevos paths para grabación local** (`RECORDINGS_DIR` con makedirs para MP4), **FFmpeg args para grabación** (`FFMPEG_RECORDING_ARGS`: H.264, CRF 23, 2Mbps, `fast` preset), **límite de grabaciones por usuario** (`MAX_RECORDINGS_PER_USER=50`), plantillas RTSP por marca (RTSP_TEMPLATES como dict para generic, hikvision, dahu, xmeye, onvif), y **JWT reforzado** (JWT_ALGORITHM='HS256', JWT_TOKEN_EXPIRATION=3600). Usa `os.getenv` y `Path` para seguridad y portabilidad.

- **`app.py`** (backend/app.py):  
  Punto de entrada principal de la aplicación Flask. Crea la instancia de app, carga configs de `config.py`, configura logging, aplica CORS con ALLOWED_ORIGINS, registra blueprints como `cameras_bp`, `auth_bp`, **`users_bp`**, **`nvr_bp`**, define rutas de monitoreo (/health y /stats para estado y métricas), maneja errores globales (400, 404, 429, **413, 507**, 500 con JSON), implementa **middleware JWT reforzado** (expiración + blacklist) aplicado **globalmente a `cameras_bp`, `users_bp` y `nvr_bp` vía `before_request`**, e inicia servicios en background como reconexión de streams via hilos. Ejecuta el servidor en host 0.0.0.0:5000 con threaded=True para concurrencia.

- **`cameras.py`** (backend/routes/cameras.py):  
  Blueprint para rutas API relacionadas con cámaras. Incluye /discover (GET, solo admin, valida network_range y credentials, llama a discover_devices y actualiza caché), /list (GET, filtra por permisos: admin ve todas, usuarios solo asignadas), /stream/<ip> (GET, inicia stream individual validando MAX_STREAMS, permisos y usando creds de caché), /streams/batch (POST, inicia múltiples streams en batch con validación de límites y permisos por IP), /stream/<ip> (DELETE, detiene stream), y /streams/<subpath> (GET, sirve archivos HLS con headers no-cache, CORS y validación de permiso por IP extraída del path). Usa `check_camera_permission(ip)` para seguridad. **Todos los endpoints requieren JWT**.

- **`users.py`** (backend/routes/users.py):  
  **Nuevo blueprint** para gestión de usuarios y asignación de cámaras. Incluye CRUD de usuarios (solo admin), asignación/desasignación de cámaras por IP, y validación de permisos. Usa `require_admin()` y `user_service.py`. **Todos los endpoints requieren JWT + rol `admin`**.

- **`nvr.py`** (backend/routes/nvr.py):  
  **Nuevo blueprint** para la función **PC-NVR** (grabación local). Integra con `recording_service.py` y `check_camera_permission(ip)`. Endpoints para iniciar/detener grabación individual o batch, listar, descargar y eliminar grabaciones. Todos protegidos por **JWT global** y permisos por IP. Admin ve todas; usuarios solo sus cámaras.

- **`video_service.py`** (backend/services/video_service.py):  
  Servicio para lógica de video. Incluye caché global (device_cache con lock), funciones para caché (get_cached_devices, update_device_cache, is_cache_valid, refresh_cache_if_needed), discover_devices (usa caché, escanea con nmap y ONVIF, expande canales DVR via _expand_dvr_channels), soporte multichannel (_expand_dvr_channels usando parse_dvr_channels y validación con ffprobe), construcción de RTSP (_construct_rtsp_url con RTSP_TEMPLATES), validación RTSP (_validate_rtsp_with_ffprobe), **start_stream con modo `record`** (RTSP → MP4 usando `FFMPEG_RECORDING_ARGS` y `RECORDINGS_DIR`), stop_stream (detiene proceso y limpia carpeta), get_active_stream_count, **get_active_recording_count()** (proxy a `recording_service`), get_stream_status, y restart_failed_streams (background para reconexión). Maneja active_streams y stream_status con locks.

- **`user_service.py`** (backend/services/user_service.py):  
  **Nuevo servicio** de lógica de negocio para gestión de usuarios. Incluye `list_users`, `get_user_detail` (con cámaras asignadas), `update_user_role`, `delete_user` (cascade SET NULL), `assign_cameras_to_user` (valida caché/DB, crea cámaras si no existen), y `remove_camera_from_user`. Usa `get_session()` y `get_cached_devices()`.

- **`recording_service.py`** (backend/services/recording_service.py):  
  **Nuevo servicio** para **PC-NVR**. Lógica core: `start_recording(ip, duration)`, `batch_start_recordings`, `stop_recording`, `get_user_recordings` (con filtros), `delete_recording_by_id`, `get_recording_status`, `get_recording_path`. Usa FFmpeg (RTSP → MP4), `generate_recording_path`, `check_disk_space`, `get_cached_devices`, `get_session`, locks (`active_recordings_lock`), monitoreo en hilo, y actualiza `Camera.is_recording` en DB.

- **`helpers.py`** (backend/utils/helpers.py):  
  Utilidades comunes. Incluye validate_network_range (valida CIDR), sanitize_ip (valida IPv4 con regex), is_valid_rtsp_url (prueba RTSP con ffprobe y timeout), generate_hls_path (crea ruta HLS segura con makedirs), **generate_recording_path** (ruta MP4 segura), **check_disk_space** (verifica espacio en disco), y parse_dvr_channels (genera lista de URLs por canal usando plantilla con {channel}).

- **`models/camera.py`** (backend/models/camera.py):  
  Modelo SQLAlchemy para cámaras. Incluye tabla 'cameras' con columnas id, ip (único), name, type, brand, channel, port, is_active, last_seen, **is_recording (Boolean, default=False)**, rtsp_url_encrypted (Text), hls_url, y **relación con User (ForeignKey)**. Métodos `set_rtsp_url` y `get_rtsp_url` para encriptar/desencriptar con Fernet usando SECRET_KEY. **Nuevos métodos**: `assign_to_user`, `remove_from_user` para gestión bidireccional segura. Usa Base de base.py.

- **`models/user.py`** (backend/models/user.py):  
  Modelo SQLAlchemy para usuarios. Incluye tabla 'users' con id, username (único), password_hash, role (Enum: 'admin', 'user', 'viewer'), created_at, y **relación uno-a-muchos con Camera** (`cameras` lista). **Nuevos métodos**: `assign_camera`, `remove_camera`, `get_assigned_camera_ips` para gestión de permisos. Soporta permisos multi-nivel.

- **`models/recording.py`** (backend/models/recording.py):  
  **Nuevo modelo** para grabaciones PC-NVR. Tabla 'recordings' con id, camera_ip, user_id (FK), start_time, end_time, file_path, duration_sec, file_size. Relación con User (`back_populates="recordings"`). Índices en `user_id + camera_ip` y `start_time`.

- **`models/base.py`** (backend/models/base.py):  
  Define `Base = declarative_base()` compartido por todos los modelos. Incluye `metadata` para migraciones futuras (Alembic).

- **`database/db.py`** (backend/database/db.py):  
  Gestión de conexión PostgreSQL. Crea engine con pool (pool_pre_ping, pool_size=10), sessionmaker, context manager `get_session()`, `init_db()` (crea tablas al startup), y funciones CRUD básicas (add_user, get_user_by_username, get_user_by_id). **Nuevas funciones CRUD para asignación**: `assign_camera_to_user`, `remove_camera_from_user`, `get_user_cameras`. **Nuevas funciones CRUD para grabaciones**: `add_recording`, `get_recordings_by_user` (con filtros), `delete_recording`. Llamado en `app.py`.

- **`auth.py`** (backend/routes/auth.py):  
  Blueprint para autenticación. Endpoints:  
  - **POST /api/auth/login**: Valida credenciales (bcrypt), genera JWT, devuelve token + user info.  
  - **POST /api/auth/logout**: Invalida token vía blacklist.  
  - **POST /api/auth/register**: Crea usuario con hashing bcrypt, **solo permite admin (o primer usuario sin token)**.  
  **No requiere JWT**.

- **`auth_service.py`** (backend/services/auth_service.py):  
  Servicio centralizado de autenticación. Incluye `validate_credentials` (bcrypt), `generate_jwt` (con expiración), `blacklist_token` / `is_token_blacklisted` (logout), `get_current_user` (opcional), y `create_user` (hashing + DB). Usa `get_session()` y modelos.

- **`auth_helpers.py`** (backend/utils/auth_helpers.py):  
  Utilidades reutilizables: `hash_password`, `verify_password` (bcrypt), `create_token`, `decode_token` (JWT wrappers). **Opcionales**, no usados actualmente pero compatibles para refactor.

- **`.env`** (raíz del proyecto):  
  Archivo ambiental con valores como FLASK_ENV=development, SECRET_KEY (fuerte), **PostgreSQL** (DB_HOST, DB_PORT, DB_USER, DB_PASS, DB_NAME), SCAN_PORTS, MAX_STREAMS=128, **RECORDINGS_DIR**, **MAX_RECORDINGS_PER_USER=50**, JWT_ALGORITHM=HS256, JWT_TOKEN_EXPIRATION=3600, etc. No versionar.

- **`requirements.txt`** (backend/requirements.txt):  
  Incluye Flask, Flask-CORS, PyJWT, bcrypt, python-nmap, onvif-zeep, wsdiscovery, python-dotenv, **SQLAlchemy**, **psycopg2-binary**, cryptography, pytest, black. Soporta PostgreSQL y autenticación segura.

## Endpoints Disponibles

Los endpoints están expuestos bajo los blueprints `/api/cameras`, `/api/users`, `/api/auth` y **`/api/nvr`**. Todos usan JSON y logging.

### Autenticación (`/api/auth`)
- **`POST /api/auth/login`**:  
  Body: `{ "username": "...", "password": "..." }` → `{ "token": "...", "user": { ... } }`.
- **`POST /api/auth/logout`**:  
  Header: `Authorization: Bearer <token>` → `{ "message": "Logged out" }`.
- **`POST /api/auth/register`**:  
  Body: `{ "username": "...", "password": "...", "role": "user" }` → crea usuario (solo admin o primer usuario).

### Usuarios (`/api/users`) **(requieren JWT + rol `admin`)**
- **`GET /api/users`**: Lista todos los usuarios (sin password_hash).
- **`GET /api/users/<id>`**: Detalle de usuario + IPs de cámaras asignadas.
- **`PUT /api/users/<id>`**: Actualiza rol. Body: `{ "role": "user" | "viewer" | "admin" }`.
- **`DELETE /api/users/<id>`**: Elimina usuario (cámaras quedan huérfanas).
- **`POST /api/users/<id>/cameras`**: Asigna cámaras. Body: `{ "camera_ips": ["192.168.0.10", ...] }`.
- **`DELETE /api/users/<id>/cameras/<ip>`**: Desasigna cámara específica.

### Cámaras (`/api/cameras`) **(requieren JWT)**
- **`GET /api/cameras/discover`**:  
  Solo admin. Parámetros: `network_range`, `credentials` → lista de dispositivos.
- **`GET /api/cameras/list`**:  
  Filtra por permisos → lista de cámaras asignadas.
- **`GET /api/cameras/stream/<ip>`**:  
  Inicia stream → `{ "hls_url": "..." }`.
- **`POST /api/cameras/streams/batch`**:  
  Body: `{ "ips": [...] }` → resultados por IP.
- **`DELETE /api/cameras/stream/<ip>`**:  
  Detiene stream.
- **`GET /api/cameras/streams/<path:subpath>`**:  
  Sirve archivos HLS (valida permiso por IP en path).

### PC-NVR (`/api/nvr`) **(requieren JWT + permiso por cámara)**
- **`POST /api/nvr/record/<ip>`**:  
  Inicia grabación individual. Body: `{ "duration": 3600 }` (opcional) → `{ "message": "Recording started", "recording_id": 123 }` (201).
- **`POST /api/nvr/records/batch`**:  
  Inicia batch. Body: `{ "ips": [...], "duration": 3600 }` → `{ "results": [{ "ip": "...", "status": "started" }] }`.
- **`DELETE /api/nvr/record/<ip>`**:  
  Detiene grabación → `{ "message": "Recording stopped" }`.
- **`GET /api/nvr/recordings`**:  
  Lista grabaciones del usuario. Query: `?camera_ip=...&start_date=...&end_date=...` → `{ "recordings": [...] }`.
- **`GET /api/nvr/recordings/<id>/download`**:  
  Descarga MP4 → stream del archivo.
- **`DELETE /api/nvr/recordings/<id>`**:  
  Elimina grabación (DB + archivo) → `{ "message": "Recording deleted" }`.
- **`GET /api/nvr/status/<ip>`**:  
  Estado en vivo → `{ "is_recording": true, "elapsed_sec": 1800 }`.

Adicionales en raíz:  
- **`GET /health`**, **`GET /stats`**.

## Funcionalidades Presentes

- **Autenticación Segura**: Login/logout con JWT (HS256, expiración), hashing bcrypt, blacklist en memoria.
- **Gestión de Cuentas y Permisos**: Roles (admin/user/viewer), asignación de cámaras a usuarios, **CRUD completo de usuarios**, **asignación/desasignación por IP**.
- **Descubrimiento de dispositivos**: Escaneo de red, multichannel DVR/NVR, caché con TTL.
- **Streaming en vivo**: RTSP → HLS con FFmpeg (baja latencia, 128 canales máx), batch, reconexión automática.
- **Función PC-NVR (Grabación Local)**: Convierte PC en NVR. RTSP → MP4, duración fija o continua, batch, listado con filtros, descarga, eliminación. Almacena metadatos en DB. Límite por usuario (`MAX_RECORDINGS_PER_USER`). Usa `Camera.is_recording` para estado en vivo.
- **Gestión de recursos**: Caché, sanitización, validación RTSP, limpieza HLS/MP4, verificación de espacio en disco (`check_disk_space`).
- **Persistencia**: **PostgreSQL** para usuarios, cámaras (encriptación RTSP), **y grabaciones**.
- **Seguridad**: JWT global en cámaras, usuarios y **NVR**, permisos por IP, CORS, headers no-cache, errores 413/507.
- **Monitoreo**: Estados de streams y grabaciones, background threads, métricas.

## Comunicación entre Componentes

La arquitectura es modular, con comunicación clara vía imports y llamadas de funciones:

- **`app.py`** orquesta: Carga `config.py`, registra `cameras_bp`, `auth_bp`, **`users_bp`**, **`nvr_bp`**, aplica **JWT global a `cameras_bp`, `users_bp` y `nvr_bp`**, inicializa DB (`init_db()`), inicia hilos de `video_service.py`.

- **`cameras.py`** llama a `video_service.py` (descubrimiento, streaming), usa `check_camera_permission` con `request.user_id` (de JWT) y modelos (`User.cameras`).

- **`users.py`** usa `user_service.py` para lógica de negocio, `require_admin()` para protección, y `get_session()` para transacciones.

- **`nvr.py`** llama a `recording_service.py` (grabación), usa `check_camera_permission(ip)` y `request.user_id` del JWT.

- **`recording_service.py`** usa `video_service.get_cached_devices()`, `generate_recording_path`, `check_disk_space`, `get_session()`, actualiza `Camera.is_recording`.

- **`user_service.py`** usa modelos (`User`, `Camera`), `get_cached_devices()` para validar IPs, y `get_session()` para persistencia.

- **`auth.py`** usa `auth_service.py` (login, register, logout) y `get_session()`.

- **`auth_service.py`** usa modelos (`User`), `bcrypt`, `jwt`, `get_session()`.

- **`video_service.py`** usa `helpers.py`, `config.py` (FFmpeg, RTSP_TEMPLATES), ejecuta procesos. Soporta modo `record`.

- **`helpers.py`** y **`auth_helpers.py`** proveen utilidades reutilizables.

- **`models/*`** y **`database/db.py`** proveen persistencia PostgreSQL. **Nuevas funciones en `db.py` para asignación y grabaciones**.

Flujo eficiente: **Autenticación → Permisos → Descubrimiento → Caché → Streaming/Grabación → Serving**, con validaciones en cada paso para robustez en 128 canales. **La gestión de usuarios y permisos está totalmente integrada con el streaming y grabación: una cámara asignada aparece en `/list`, permite `/stream/<ip>` y `/record/<ip>` inmediatamente.**
