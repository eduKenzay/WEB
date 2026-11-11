# ANTRAXCRAFT

Portal de Minecraft Java Edition 1.21.1 con NeoForge. Desarrollado con Next.js App Router, TypeScript, Tailwind CSS, Motion y Lucide.

## Desarrollo local

Usa Node.js 24 LTS y pnpm 11.25.0.

```sh
npm install -g pnpm@11.25.0
pnpm install --frozen-lockfile
cp .env.example .env.local
pnpm dev
```

En PowerShell, reemplaza `cp .env.example .env.local` por `Copy-Item .env.example .env.local`. Abre http://localhost:3000.

```sh
pnpm lint
pnpm typecheck
pnpm build
pnpm start
```

## Configuración

Los valores se leen en el servidor desde `src/config/site.ts`. Para aplicar cambios a la portada o metadatos en producción, vuelve a ejecutar el build y reinicia el proceso.

| Variables | Uso |
| --- | --- |
| `SITE_URL` | Origen HTTPS público. Activa URLs absolutas para Open Graph y el sitemap. |
| `PROJECT_NAME`, `MINECRAFT_VERSION`, `NEOFORGE_VERSION` | Identidad y versiones visibles. La revisión exacta de NeoForge es opcional. |
| `SHOW_WELCOME` | Cambia a `true` para mostrar la bienvenida una vez por sesión. |
| `MINECRAFT_HOST`, `MINECRAFT_PORT`, `MINECRAFT_PROTOCOL` | Host o IP de consulta, puerto TCP y protocolo Java. Los valores por defecto son 25565 y 767 para Minecraft 1.21.1. Configura el destino real si usas un registro SRV. |
| `MINECRAFT_PUBLIC_ADDRESS` | Dirección que puede ver el visitante. Es independiente del host de consulta. |
| `LAUNCHER_URL`, `LAUNCHER_VERSION`, `LAUNCHER_SIZE`, `LAUNCHER_UPDATED` | URL HTTPS del instalador, versión, tamaño legible y fecha AAAA-MM-DD. Con la URL vacía, el botón aparece deshabilitado. |
| `DISCORD_URL`, `FACEBOOK_URL`, `SUPPORT_URL` | Enlaces HTTPS oficiales. Si faltan, sus espacios se muestran como próximos. |

En `src/content/site-content.ts` puedes actualizar el estado de las funcionalidades, la guía de instalación, requisitos, notas de versión y las noticias destacadas. Completa los textos de privacidad y términos según los datos del operador y las prácticas del alojamiento antes de publicar.

## Noticias

Añade un archivo Markdown en `src/content/news/` con un nombre como `nueva-actualizacion.md`. La ruta pública será `/noticias/nueva-actualizacion`.

```yaml
---
title: "Título de la publicación"
date: "2026-11-10"
category: "Actualizaciones"
image: "/art/valle.webp"
summary: "Resumen de la publicación."
demo: false
---
```

Las categorías admitidas son Actualizaciones, Eventos, Desarrollo, Comunidad y Mantenimiento. Las imágenes se guardan en `public/art/` en WebP. Los tres artículos iniciales se identifican como demostraciones y quedan fuera del sitemap.

## Panel de administración y catálogo

El acceso del equipo está en `/admin`, también enlazado desde el pie de página. Permite crear y editar noticias, jefes finales y biomas, guardar borradores, publicar, retirar una entrada de publicación y seleccionar destacados. Los jefes finales incluyen un campo de mundo obligatorio. El catálogo público permite filtrar por categoría y mundo.

Configura la contraseña desde una terminal privada. El asistente escribe el hash y un secreto de sesión aleatorio en el archivo de entorno, sin imprimirlos. La contraseña se ve mientras la escribes en esa terminal.

```sh
# En desarrollo
node scripts/configure-admin.mjs
# En el VPS de producción
node scripts/configure-admin.mjs --production
```

Después reinicia Next.js. Las sesiones duran ocho horas. Cambiar `ADMIN_SESSION_SECRET` invalida todas las sesiones. El acceso queda cerrado hasta configurar ambas variables. Usa HTTPS en producción. El login limita a diez intentos por proceso cada quince minutos.

Los cambios se guardan como JSON mediante escritura atómica, con detección de conflictos de edición. Los archivos originales Markdown sirven como base. Las páginas públicas leen el catálogo al solicitarse, de modo que las publicaciones del panel aparecen sin recompilar.

En el VPS configura una carpeta persistente fuera de los despliegues. Por ejemplo, crea `/srv/antraxcraft-data/articles` con propietario `antraxcraft` y permisos `700`, y define `CONTENT_DIRECTORY=/srv/antraxcraft-data/articles`. Conserva copias de seguridad de esa carpeta. En local se usa `data/articles`, excluida de Git. Todos los procesos de la web deben usar el mismo directorio.

El selector de portadas ofrece las ilustraciones del proyecto. Para añadir una imagen nueva, colócala en `public/art` y registra su ruta en `src/content/editorial.ts`. El panel actual usa una contraseña compartida para el equipo y puede evolucionar a usuarios y roles con MariaDB.

Si el proceso se interrumpe durante un guardado y queda un archivo `.lock`, detén las instancias de la aplicación antes de retirar ese archivo de la carpeta de contenido. Luego reinicia la web y vuelve a guardar.

## Capas de la aplicación

- `src/app/` contiene las rutas, la metadata y el endpoint `GET /api/status`.
- `src/components/` reúne la interfaz compartida y los componentes interactivos.
- `src/config/` valida y separa la configuración privada de la información que llega al navegador.
- `src/content/` contiene la bitácora, los pasos del launcher y el catálogo Markdown.
- `src/services/` contiene el repositorio editorial, el ping TCP Java y los contratos de perfiles y vinculación.
- `public/brand/` conserva copias exactas de los PNG adjuntos. `public/art/` contiene arte conceptual WebP generado para el portal.

`GET /api/status` consulta exclusivamente el host configurado, con un plazo de tres segundos, caché de 30 segundos, deduplicación de solicitudes y un límite global por proceso. La respuesta no publica el host de consulta ni la lista de jugadores. Una conexión rechazada se muestra como fuera de línea. Un fallo de red, de protocolo o una configuración vacía se muestra como estado desconocido.

La interfaz `NewsRepository` permite sustituir Markdown por un repositorio MariaDB. El futuro mod NeoForge podrá integrarse mediante una API autenticada y los contratos de jugadores. El portal se ejecuta sin base de datos ni credenciales del juego.

## Despliegue en Ubuntu 24.04 con Nginx y HTTPS

En un VPS con Ubuntu 24.04 puedes usar la web completa, incluido el panel editorial. Necesitas acceso SSH con sudo y un dominio cuyo DNS apunte al VPS. Permite TCP 80 y 443 desde Internet y la salida TCP al puerto de Minecraft si usarás el indicador de estado. Mantén el acceso SSH habilitado.

1. Sube el código del proyecto a una carpeta temporal de tu usuario SSH. Incluye `pnpm-lock.yaml`, `src/`, `public/`, `scripts/` y `deploy/`. Excluye `node_modules/`, `.next/`, los archivos `.env*` privados y `data/`; conserva `.env.example`. No reutilices dependencias de Windows en Linux.

2. Desde la copia temporal, instala las dependencias del VPS y copia los archivos a su ubicación definitiva:

   ```sh
   sudo bash deploy/ubuntu/bootstrap.sh
   sudo cp -a . /srv/antraxcraft/
   sudo chown -R antraxcraft:antraxcraft /srv/antraxcraft
   cd /srv/antraxcraft
   ```

   El instalador prepara Node.js 24, pnpm 11.25.0, Nginx, Certbot y el directorio persistente de publicaciones. Para Node.js usa el [repositorio de NodeSource para Ubuntu](https://github.com/nodesource/distributions/blob/master/DEV_README.md).

3. En el primer despliegue, configura el archivo de entorno y crea tu acceso al panel:

   ```sh
   sudo -u antraxcraft install -m 0600 .env.example .env.production.local
   sudo -u antraxcraft nano .env.production.local
   sudo -u antraxcraft /usr/bin/node scripts/configure-admin.mjs --production
   ```

   Escribe `SITE_URL=https://antraxcraft.com` y descomenta `CONTENT_DIRECTORY=/srv/antraxcraft-data/articles`. Configura el servidor de Minecraft y la descarga del launcher si ya tienes sus datos. Conserva el archivo de entorno al actualizar.

4. Instala las dependencias exactas del proyecto, compila y habilita el servicio:

   ```sh
   sudo -u antraxcraft -H pnpm install --frozen-lockfile --prod=false
   sudo -u antraxcraft -H pnpm build
   sudo install -m 0644 deploy/ubuntu/antraxcraft.service /etc/systemd/system/antraxcraft.service
   sudo systemctl daemon-reload
   sudo systemctl enable --now antraxcraft
   curl -fsS http://127.0.0.1:3000/api/status
   ```

5. Crea un registro DNS de tipo A para `antraxcraft.com` que apunte a `103.195.103.74`. Instala la configuración de Nginx y activa HTTPS cuando el registro ya resuelva a la IP del VPS:

   ```sh
   sudo install -m 0644 deploy/ubuntu/antraxcraft.nginx.conf /etc/nginx/sites-available/antraxcraft
   sudo ln -s /etc/nginx/sites-available/antraxcraft /etc/nginx/sites-enabled/antraxcraft
   sudo nginx -t && sudo systemctl reload nginx
   sudo /snap/bin/certbot --nginx --redirect -d antraxcraft.com
   sudo /snap/bin/certbot renew --dry-run
   curl -fsS https://antraxcraft.com/api/status
   ```

Abre la web y comprueba el acceso a `/admin`. Después podrás publicar desde el navegador. Haz copias de seguridad de `/srv/antraxcraft-data/articles` y de la configuración privada. Para actualizar el código, conserva ambos, vuelve a instalar y compilar y reinicia el servicio con `sudo systemctl restart antraxcraft`. Si el servicio falla, revisa `sudo journalctl -u antraxcraft -n 50`.

[Next.js en servidor propio](https://nextjs.org/docs/app/guides/self-hosting) · [Certbot para Nginx](https://certbot.eff.org/instructions?os=snap&ws=nginx)

## Referencia visual

OneRol presenta una entrada opcional con una transición a su portada, navegación fija por anclas y recursos que evocan su mundo, como un expediente paginado. ANTRAXCRAFT usa esos principios con una bienvenida configurable, un paisaje propio y una bitácora de pergamino.

[OneRol](https://onerol.net)

## Arte conceptual

Las ilustraciones y el mapa conceptual se generaron con ImageGen integrado, sin texto ni marcas.

- `public/art/mapa-expedicion.webp` — *Create a 16:9 wide full-bleed background for the hero of a medieval voxel fantasy video game website: an authentic antique expedition map drawn in burnt umber ink on weathered golden ochre parchment, with a cinematic adventurous archaeological adventure serial feeling. Top-down illustrated cartography: irregular coastlines, tight engraved contour lines, forest clusters, winding rivers, tiny angular mountain ridges, subtle medieval tower and ruins symbols. A few purposeful dashed deep red travel routes linking sites, with one X near lower left and one near lower right, compass rose upper right away from the edge. Rich warm honey gold, terracotta ink and patches of muted moss green. Visually rich and intricately detailed toward left and right thirds, with softer sparse map details in the central third so a separately rendered game logo and overlay text remain dominant. Terrain, rivers and roads should be visible across the WHOLE image including the middle; no blank frame, no objects sitting on the parchment, no giant compass, no decorative outer border. Overhead flat art, natural paper grain and dark burnt vignette at edges. No words, no lettering, no typography, no logo, no people, no photographs. The map itself fills the image edge to edge.*

- `public/art/valle.webp` — *Ultra wide 16:9 cinematic high fidelity Minecraft voxel medieval fantasy promotional landscape, dramatic grand stone castle city with emerald slate rooftops on far RIGHT side, two vivid icy cyan waterfalls cascading down stepped cliffs, dense emerald pine forests in foreground left, center open misty valley with a winding river, sunset sunlight breaking through clouds upper left, distant mountains. Bright turquoise teal atmosphere with warm amber sunlight. Clear depth, finely detailed square blocks, physically based 3D game render, polished official game key art, no text, no logo, no UI.*
- `public/art/portal.webp` — *Wide landscape game concept art, Minecraft style authentic square voxel block world, a mossy ancient gothic stone arch with brilliant turquoise magical portal and stepped stone approach in a deep forest ravine, emerald green lush foliage and roots, amber lanterns, crystalline cyan mist and streaming sunlight, epic fantasy adventure environment, high end cinematic 3D game art, no text or logos.*
- `public/art/aldea.webp` — *Wide landscape polished Minecraft voxel block medieval fantasy scene, cozy inhabited medieval trading village with timber frame homes, warm amber windows, wheat fields, colorful market awnings, stepped hills and blue mountains, golden hour sunshine, beautiful cinematic depth, authentic block geometry. No text no logos.*
- `public/art/mazmorra.webp` — *Wide Minecraft voxel cinematic dark fantasy landscape, underground ancient stepped stone dungeon bridge crossing a deep cave, brilliant amethyst purple geode crystals and luminous purple portal beyond, mist, copper lanterns and dark granite stone, game key art, no text no logos.*

