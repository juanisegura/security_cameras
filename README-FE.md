# Frontend para Sistema de Cámaras de Seguridad

Este README describe el desarrollo del frontend para un sistema de cámaras de seguridad. El frontend está construido en **PHP** para manejar la lógica de servidor, sesiones y redirecciones, integrado con HTML, CSS y JavaScript para la interfaz de usuario. El enfoque principal es **forzar un login obligatorio** antes de mostrar cualquier contenido sensible, como las cámaras en vivo. Solo después de una autenticación exitosa se muestra el dashboard con visualización de hasta **128 canales**, gestión de layouts, y otras funcionalidades.

El frontend se comunica con el backend (Flask/Python) a través de APIs REST para autenticación, listado de cámaras, streams HLS y grabaciones PC-NVR. Se usa **sesión PHP** para manejar el estado de autenticación y **JWT** para llamadas seguras al backend.

## Objetivos del Frontend
- Proporcionar una interfaz amigable para **login/logout**.
- **Visualización en vivo** de cámaras IP/DVR/NVR (hasta 128 canales), con selección de layout (1x1, 2x2, 4x4, etc.).
- Soporte para **PC-NVR**: Grabación local en el PC del usuario.
- **Gestión de cuentas y permisos**: Diferentes roles (admin, viewer) con accesos limitados.
- **Seguridad**: Nada visible sin login; uso de sesiones y tokens.
- Interfaz responsiva y drag-and-drop para ordenar cámaras.

## Estructura de Archivos
A continuación, se lista la estructura de carpetas y archivos, junto con la lógica que contiene cada uno.

### Raíz (`frontend/`)
- **index.php**: Punto de entrada principal. Verifica la sesión: si autenticado, incluye `dashboard.php`; si no, incluye `login.php`. Maneja el esqueleto HTML base (incluyendo `header.php` y `footer.php`).
- **.htaccess**: Configura reglas de rewrite para rutas limpias, protege directorios sensibles y fuerza HTTPS en producción.

### Configuración (`config/`)
- **config.php**: Constantes globales como URL del backend (e.g., `API_BASE_URL = 'http://localhost:5000/api';`), claves de sesión y configuraciones de entorno (dev/prod).
- **auth.php**: Funciones de autenticación: validar token, chequear roles, manejar blacklist de tokens (si aplica).

### Includes (`includes/`)
- **header.php**: Contiene el `<head>` con meta tags, enlaces CSS, scripts JS globales (e.g., hls.js CDN) y cualquier lógica PHP para inyectar variables en JS (e.g., token).
- **footer.php**: Cierre del `<body>`, scripts JS al final para optimización, y cualquier código de tracking/analytics.
- **functions.php**: Utilidades generales: funciones para llamadas API (cURL o fetch), formateo de fechas, manejo de errores y helpers para HLS streaming.

### Páginas (`pages/`)
- **login.php**: Muestra solo el formulario de login (username, password). Lógica: procesa POST, llama a `/api/login.php`, guarda sesión si OK, redirige a dashboard. Muestra errores si falla.
- **dashboard.php**: Dashboard principal (visible solo post-login). Incluye grid de cámaras, sidebar de dispositivos, barra de funciones (layouts). Lógica: verifica sesión, carga cámaras vía JS, inicia streams HLS.
- **logout.php**: Procesa logout: destruye sesión, revoca token (llamada a backend), redirige a `/`.

### Assets (`assets/`)
#### CSS (`assets/css/`)
- **login.css**: Estilos para el formulario de login (centrado, minimalista, responsive).
- **dashboard.css**: Estilos para el dashboard: grid flexible para cámaras, sidebar, top-bar con logout.
- **hls-player.css**: Estilos específicos para videos HLS (overlays de loading/error, labels de cámara).

#### JS (`assets/js/`)
- **auth.js**: Lógica de autenticación en cliente: valida formulario login, maneja AJAX a `/api/login.php`, actualiza UI post-login (muestra/oculta elementos), chequea token expirado.
- **cameras.js**: Carga lista de cámaras del backend, inicia streams HLS batch, maneja reproducción con hls.js. Lógica: fetch con token, actualización dinámica del grid.
- **grid.js**: Manejo de layouts: cambia columnas/rows (e.g., 1x1 a 4x4), drag-and-drop para reordenar cámaras. Lógica: eventos DOM, almacenamiento local para preferencias.
- **hls.js**: Wrapper para hls.js library: attach/detach media, manejo de errores en streams.

#### Images (`assets/images/`)
- **logo.png**: Logo del sistema.
- **icons/**: Íconos para sidebar (cámaras, grabaciones, usuarios) y botones.

### API Interna (`api/`)
Estos son archivos PHP que actúan como proxies o endpoints internos para el frontend, manejando llamadas al backend real.
- **login.php**: Procesa POST de login: envía a backend `/api/auth/login`, guarda token en sesión, retorna JSON (success/error).
- **cameras.php**: GET para listar cámaras: llama a backend `/api/cameras/list`, filtra por rol, retorna JSON para JS.
- **streams.php**: POST para iniciar streams: envía IPs a backend `/api/cameras/streams/batch`, retorna URLs HLS.

## APIs a Crear (Endpoints del Backend Consumidos por el Frontend)
El frontend consume estas APIs REST del backend (Flask). Se asume que ya están implementadas en el backend; el frontend las llama con `Authorization: Bearer <token>`.

- **/api/auth/login** (POST): Autenticación. Body: `{username, password}`. Respuesta: `{token, user: {id, username, role}}`.
- **/api/auth/logout** (POST): Revoca token (agrega a blacklist). Respuesta: `{message: 'Logged out'}`.
- **/api/users** (GET/POST/PUT/DELETE): Gestión de usuarios (solo admin). Ej: GET lista usuarios, POST crea nuevo.
- **/api/cameras/discover** (GET/POST): Escanea red para cámaras. Params: `network_range`, body: `{credentials: {username, password}}`.
- **/api/cameras/list** (GET): Lista cámaras asignadas al usuario. Respuesta: `[{ip, name, type}]`.
- **/api/cameras/stream/<ip>** (GET): Inicia stream HLS para una cámara. Respuesta: `{hls_url}`.
- **/api/cameras/streams/batch** (POST): Inicia múltiples streams. Body: `{ips: []}`. Respuesta: `{results: [{ip, hls_url, status}]}`.
- **/api/nvr/record** (POST): Inicia grabación PC-NVR. Body: `{ip, duration}`. Respuesta: `{recording_id, path}`.
- **/api/nvr/list** (GET): Lista grabaciones del usuario. Respuesta: `[{id, ip, timestamp, path}]`.
- **/api/nvr/download/<id>** (GET): Descarga grabación (stream o link).

## Instrucciones de Instalación y Ejecución
1. Configura PHP (7.4+) con servidor (Apache/Nginx) o usa `php -S localhost:3000`.
2. Actualiza `config.php` con `API_BASE_URL`.
3. Asegura PostgreSQL y backend corriendo en puerto 5000.
4. Accede a `http://localhost:3000` → verás solo login.
5. Login con `admin/admin123` → dashboard con cámaras.

## Notas de Desarrollo
- **Seguridad**: Todas las llamadas API usan token JWT; sesiones PHP expiran en 1h.
- **Dependencias**: hls.js (CDN), jQuery (opcional para AJAX).
- **Expansión**: Agregar roles: admin ve gestión usuarios, viewer solo visualización.
- **Testing**: Usa Postman para APIs; browser dev tools para JS.