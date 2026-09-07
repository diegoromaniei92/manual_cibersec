# Política de Seguridad — Manual CiberSec (CMS Ciberseguridad)

Este documento describe los **protocolos de seguridad** implementados en el sitio
y las prácticas recomendadas para mantenerlo seguro. El sitio es una aplicación
de una sola página (SPA) que almacena los datos localmente en el navegador
(`localStorage`) y embebe videos de YouTube.

---

## 1. Modelo de amenazas

| Amenaza | Vector | Mitigación implementada |
|---|---|---|
| **XSS (Cross-Site Scripting)** | Inyección de HTML/JS mediante títulos, descripciones o URLs de videos (datos del usuario en `localStorage`) | Escape de todo contenido dinámico con `escapeHtml()`; validación estricta de IDs de YouTube (`^[a-zA-Z0-9_-]{11}$`); saneo del esquema al cargar datos; eliminación de manejadores `onclick` inline |
| **Exfiltración de datos** | `data:` URIs, formularios externos, carga de scripts de orígenes desconocidos | **CSP** restringe `script-src`, `style-src`, `img-src`, `frame-src`, `form-action`, `object-src 'none'` y `base-uri 'self'` |
| **Clickjacking / embebido malicioso** | El sitio es embebido en `iframe` de terceros con capas ocultas | `X-Frame-Options: DENY` + `frame-ancestors` (en encabezados) |
| **Robo de credenciales/CSRF** | Aprovechar cookies o sesiones | El sitio **no usa cookies ni sesiones**; no hay autenticación. `form-action 'self'` limita envíos de formularios |
| **MitM / HTTP plano** | Interceptación de tráfico | `upgrade-insecure-requests` + `Strict-Transport-Security` (HSTS) + forzado HTTPS vía `.htaccess` |
| **Abuso de APIs del dispositivo** | Cámara, micrófono, geolocalización, pago | `Permissions-Policy` deniega todas las funciones no necesarias |
| **Manipulación de `localStorage`** | Usuario/extensiones inyectan datos malformados | Saneo y validación del esquema en `loadData()` |

## 2. Controles implementados

### 2.1 Content Security Policy (CSP)
Definida como `<meta http-equiv="Content-Security-Policy">` (GitHub Pages no permite
encabezados HTTP personalizados) y repetida en `_headers` (Netlify/Cloudflare Pages)
y `.htaccess` (Apache).

Directivas clave:
- `script-src` limitada a `'self'`, Tailwind CDN, YouTube y Cloudflare Insights.
- `frame-src` limitada a `https://www.youtube.com` y `https://www.youtube-nocookie.com` (nada más puede embeberse).
- `object-src 'none'`, `base-uri 'self'`, `form-action 'self'`.
- `upgrade-insecure-requests`: toda subpetición se fuerza a HTTPS.

> **Nota:** `'unsafe-inline'`/`'unsafe-eval'` en `script-src` son necesarios por el
> **Tailwind Play CDN** (`cdn.tailwindcss.com`), que compila clases en tiempo de
> ejecución. *Endurecimiento recomendado:* migrar a una build local de Tailwind
> (usa `@tailwindcss/cli`, genera el `index.html` con clases ya compiladas y
> elimina ese CDN). Eso permite quitar `'unsafe-inline'` y `'unsafe-eval'` y
> usar nonces + `strict-dynamic`. Ver sección 5.

### 2.2 Sin manejadores inline y escape de salida
- Todos los eventos se manejan por **delegación** (`data-action` + `addEventListener`), eliminando los `onclick` inline.
- Toda interpolación de datos del usuario en el DOM pasa por `escapeHtml()`.
- Los IDs de YouTube solo aceptan el conjunto `[a-zA-Z0-9_-]{11}`.

### 2.3 Encabezados
- `X-Content-Type-Options: nosniff`
- `X-Frame-Options: DENY`
- `Referrer-Policy: strict-origin-when-cross-origin`
- `Permissions-Policy:` cámara, micrófono, geolocalización, pago y USB denegados.
- `Cross-Origin-Opener-Policy: same-origin`
- `Cross-Origin-Resource-Policy: same-origin`
- `Strict-Transport-Security: max-age=63072000; includeSubDomains; preload`

> Para GitHub Pages, los encabezados que no pueden emitirse por HTTP se compensan
> con la CSP meta y el aislamiento por navegador (no hay cookies, no hay estado
> de servidor, no hay autenticación).

## 3. Protocolos de despliegue

1. **Sirve siempre por HTTPS** (GitHub Pages lo hace por defecto).
2. Si usas **PayPal/GitHub Pages**: los encabezados extra van en `_headers` (Cloudflare Pages/Netlify) o `.htaccess` (Apache). GitHub Pages usará la CSP del `<meta>`.
3. **No subas secretos.** Este proyecto no debe contener tokens, API keys o credenciales. Si algún día se agrega backend/autenticación:
   - Almacena secretos fuera del repo (variables de entorno).
   - Usa cookies `HttpOnly`, `Secure`, `SameSite=Strict`.
   - Valida y sanitiza *todo* input de servidor (nunca confíes en el cliente).
   - Aplica rate limiting y captura errores sin filtrar detalles internos.
   - Usa cabecera CSP generada por servidor con nonces.
4. **Mantén actualizada la lista de dominios autorizados** en la CSP si agregas fuentes (por ejemplo, nuevos CDN de video—evita `googleusercontent.com` o `googlevideo.com` en `frame-src`; solo se permiten `youtube.com` y `youtube-nocookie.com`).
5. **Auditoría periódica**: revisa `git log` y el historial de colaboradores; habilita protección de rama `main` en GitHub y revisa los permisos del repositorio.

## 4. Datos en el dispositivo

- Los videos/modulos se guardan solo en `localStorage` del navegador (**no hay backend**).
- Esto significa que el contenido es editable por cualquier persona con acceso al navegador.
- Si necesitas ediciones persistentes, colaborativas o con control de acceso, se requiere un **backend con autenticación** (fuera de alcance de este sitio estático).

## 5. Endurecimiento recomendado (roadmap)

1. **Quitar el Tailwind Play CDN** y compilar `index.html` a estáticos:
   ```bash
   npm i -D tailwindcss @tailwindcss/cli
   npx tailwindcss -i custom.css -o assets/tailwind.css --minify
   ```
   Luego: `script-src 'self'` + nonces, quitar `'unsafe-inline'` y `'unsafe-eval'`.
2. **Autoalojar FontAwesome** (`@fortawesome/fontawesome-free`) para eliminar el CDN de cdnjs de la CSP.
3. **Integrar un Origin Shield / WAF** (Cloudflare) con el sitio desplegado.
4. **Añadir encabezados HTTP** reales si el hosting lo soporta (con `_headers`/`.htaccess` ya incluidos).

---

*Si encuentras una vulnerabilidad, por favor contacta al mantenedor y NO la divulguemos*
*en issues públicos.*