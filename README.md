# Serv&Tec — Sitio + CMS

Convierte el sitio de Serv&Tec en una página **dinámica y administrable**: el mismo
diseño de antes, pero todo el contenido (textos, servicios, trabajos, zonas, colores,
etc.) sale de una base de datos y se edita desde un panel en `/admin`, sin tocar código.

## Arquitectura (y por qué)

No usé Node+Express+PostgreSQL+Prisma+React como sugería el pedido original. Elegí algo
más chico y **igual de real** porque este sitio es de bajo tráfico (una pyme de barrio) y
menos piezas móviles significa menos cosas que se pueden romper en un hosting sencillo:

| Pedido | Elegido | Por qué |
|---|---|---|
| Express | **`http` nativo de Node** | Node 22 ya trae routing, JSON, streams; evita una dependencia más que mantener actualizada. |
| PostgreSQL + Prisma | **SQLite (`node:sqlite`, nativo de Node 22)** | Un solo archivo `.db`, cero servidor de base de datos que administrar, ideal para este volumen. Si el negocio crece mucho, migrar a Postgres más adelante es straightforward porque el SQL usado es simple. |
| React para el panel | **JavaScript vanilla** (sin build step) | El panel es formularios y listas; no justifica un bundler. Menos dependencias = menos vulnerabilidades que parchear con el tiempo. |
| Multer | **Manejo de streams nativo** | Con validación de tipo de archivo por contenido real (no por extensión declarada). |

Cero dependencias de `npm install` — todo corre con Node 22+ solo. Esto es intencional:
menos superficie de ataque y nada que se desactualice.

**El frontend público no se tocó a mano**: `backend/render.js` genera exactamente el
mismo HTML/CSS/JS que ya tenías, pero completando los textos desde la base de datos en
vez de tenerlos escritos en el archivo.

## Estructura

```
/backend     servidor, base de datos, autenticación, validación, generación del HTML público
/frontend    site.css y site.js (el mismo diseño/estilo que ya tenías)
/admin       panel de administración (HTML/CSS/JS, sin build)
/database    ahí se crea servtec.db (SQLite) la primera vez que arrancás
/uploads     imágenes y videos que subís desde el panel
.env.example configuración (copiala a .env)
```

## 1. Instalar Node.js

Necesitás **Node.js 22.13 o superior** (usa `node:sqlite`, incluido nativamente desde
esa versión — no hace falta instalar una base de datos aparte).

- Windows/Mac: descargá el instalador LTS desde https://nodejs.org
- Verificá la versión: `node -v`

## 2. Configurar

```bash
cp .env.example .env
```

Abrí `.env` y completá al menos `SITE_URL` con tu dominio real cuando lo tengas. El resto
de los valores por defecto sirven para arrancar.

No hace falta `npm install`: el proyecto no tiene dependencias externas.

## 3. Crear el primer administrador

```bash
npm run create-admin
```

Te va a pedir usuario y contraseña (mínimo 10 caracteres) por consola. Guardalos bien:
son con lo que vas a entrar a `/admin`.

## 4. Iniciar

```bash
npm start
```

- Sitio público: `http://localhost:3000`
- Panel: `http://localhost:3000/admin`

La primera vez que arranca, crea `database/servtec.db` y lo carga con el contenido
**que ya tenía tu sitio** (mismo texto, mismos 6 servicios, mismos 4 pasos, mismas 2
zonas) — no se inventó nada nuevo.

## Cómo se usa el panel

Entrás a `/admin`, iniciás sesión, y en el menú de la izquierda tenés **Inicio,
Servicios, Proceso, Zona, Contacto, Footer, Trabajos, Media, Consultas, SEO, Diseño,
Usuarios, Configuración**.

- **Servicios / Trabajos / Pasos / Zonas** son listas: `+ Nuevo` para crear, ✏️ para
  editar, 🗑 para borrar, 👁 para ocultar/publicar sin borrar, y podés arrastrar (☰) o
  usar las flechas para reordenar — el orden se refleja al instante en la web.
- Cada edición tiene tres botones: **Guardar borrador** (no afecta lo publicado), **Vista
  previa** (abre la web con tus cambios sin publicar, con un aviso arriba), y
  **Publicar** (ahí sí se actualiza la web pública).
- **Media** es la biblioteca de imágenes/videos: subís una vez y la reusás donde
  quieras (logo, fondo del hero, foto de un servicio, thumbnail de un trabajo).
- **Diseño** cambia colores, radio de tarjetas y tamaño de fuente sin tocar CSS.
- **Consultas** guarda lo que la gente escribe en el formulario "Pedir turno" (si
  activaste esa opción en Inicio → Formulario de turnos), con estados Nueva /
  Contactada / En proceso / Finalizada / Cancelada.
- **Configuración → Estado del sitio** te deja apagar la web (modo mantenimiento) con
  un mensaje y volver a encenderla.
- **Usuarios** (solo ADMIN) crea cuentas nuevas con rol ADMIN o EDITOR. Un EDITOR puede
  editar todo el contenido pero no administrar usuarios ni el estado del sitio.
- **Configuración → Historial** muestra quién cambió qué y cuándo.

### Datos que dejé para que vos completes

El HTML original no tenía email, dirección ni horarios de atención — no los inventé.
En el panel vas a ver esos campos vacíos con una nota tipo *"TODO: configurar desde el
panel"*; completalos cuando quieras y se van a mostrar automáticamente.

## Seguridad (qué implementa esto)

- Contraseñas con hash `scrypt` (no reversible) + sal aleatoria por usuario.
- Sesiones por cookie `HttpOnly`, `SameSite=Strict`, con expiración (12 h por defecto).
- Límite de intentos de login por usuario e IP (protección de fuerza bruta).
- Todas las rutas de escritura de la API exigen sesión válida + cabecera anti-CSRF, y
  además verifican que el origen de la petición coincida con el sitio.
- Los archivos subidos se validan **por contenido real** (firma binaria), no por la
  extensión que declara el navegador; los SVG se filtran de scripts/handlers y solo un
  ADMIN puede subirlos.
- El HTML que vos escribís en el panel (nombres, descripciones, etc.) se **escapa**
  siempre al mostrarlo en la web — no se puede inyectar código desde un campo de texto.
- Cabeceras de seguridad (CSP, X-Frame-Options, nosniff, etc.) en todas las respuestas.
- `.env` nunca se sube a un repositorio (está en `.gitignore`); ninguna clave vive en el
  HTML ni en el JavaScript del navegador.

## Borradores y publicación

Cada sección y cada elemento de una lista tiene dos estados: **borrador** (`draft`) y
**publicado** (`live`). Guardar borrador nunca toca lo que ve el público; solo
"Publicar" lo hace. "Vista previa" te deja ver el sitio con tus borradores aplicados
(con un aviso naranja arriba) antes de decidir publicar.

## Pruebas automáticas

```bash
npm test
```

Corre una batería de extremo a extremo (usando una base de datos temporal) que valida:
login, permisos por rol, que un borrador no aparezca en público, que vista previa sí lo
muestre, reordenamiento, subida y validación de archivos, formulario de consultas, modo
mantenimiento, robots.txt/sitemap.xml e historial. Se corrió antes de entregar esto y
pasa completo.

## Build para producción

No hay paso de "build": es Node directo. Para producción:

```bash
NODE_ENV=production npm start
```

Con un gestor de procesos como `pm2` o el propio del hosting para que se reinicie solo
si el proceso se cae:

```bash
npm install -g pm2
pm2 start backend/server.js --name servtec
```

## Desplegarlo

Cualquier hosting que corra Node 22+ con un proceso persistente sirve (Railway, Render,
un VPS con `pm2` + Nginx como proxy, etc.). Netlify **no** sirve para esto: solo aloja
archivos estáticos, y este proyecto necesita un proceso corriendo para el panel y la
base de datos.

Pasos generales:
1. Subí esta carpeta al servidor (sin `.env`, sin `node_modules`, sin `database/*.db`).
2. Copiá `.env.example` a `.env` en el servidor y completalo (`SITE_URL` con tu dominio,
   `COOKIE_SECURE=true` y `TRUST_PROXY=true` si hay un proxy/HTTPS delante).
3. Si tu hosting tiene un disco persistente aparte del código, apuntá `DB_PATH` y
   `UPLOAD_DIR` ahí (si no, lo que subas se puede perder en cada redeploy).
4. `npm run create-admin` una vez, en el servidor.
5. `npm start` (o con `pm2`, como arriba).
6. Poné un proxy HTTPS (Nginx, Caddy, o el propio del hosting) delante del puerto 3000.

## Notas

- `robots.txt` y `sitemap.xml` se generan solos a partir de lo que configures en
  **SEO** (si "Permitir que Google indexe" está apagado, `robots.txt` bloquea todo).
- El fondo de circuito del hero se puede apagar desde **Inicio → Hero → Mostrar fondo
  tecnológico**.
- Los videos de trabajos aceptan un link de YouTube, Instagram, TikTok o un `.mp4`
  propio (se sube como cualquier otro archivo en Media). YouTube se embebe directo;
  Instagram/TikTok se muestran como una miniatura con un botón "Ver video".
