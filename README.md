# Consule - Despliegue con HTTPS (Nginx reverse proxy + frontend)

Este repositorio está organizado como monorepo:
- `consulecei-frontend/`: Aplicación Angular, con su `Dockerfile` listo.
- `backend/`: (Opcional) Servicio backend. No se incluye configuración de Docker en este momento.
- `reverse-proxy/`: Nginx que termina TLS (HTTPS) para tu dominio y hace proxy al `frontend` por red interna.

Tu dominio consulecei.com ya apunta a tu servidor. Este stack publica 80/443 en el host, sirve el certificado de Let's Encrypt (que tú emitirás manualmente en el servidor) y reenvía el tráfico a `frontend` (HTTP interno, no expuesto).

---

## Arquitectura
- Cliente → HTTPS (443) → Nginx reverse-proxy → HTTP interno → `frontend`
- Puerto 80 solo se usa para:
  - Resolver los retos de ACME (/.well-known/acme-challenge/) si usas certbot con método `--webroot`.
  - Redirigir el resto del tráfico a HTTPS.

## Requisitos en el servidor
- Docker 24+ y Docker Compose v2 (`docker compose version`).
- DNS del dominio apuntando a tu servidor (A/AAAA): consulecei.com → 72.60.251.88
- Certbot instalado para emitir/renovar certificados (lo ejecutarás en el host).
- Puertos 80 y 443 abiertos en el firewall del servidor.

## Paso 1: Variables de entorno
En la raíz del repo:

```
cp .env.example .env
```

Edita `.env` y ajusta:
- `DOMAIN`: tu dominio (por defecto ya está `consulecei.com`).
- `API_BASE_URL`: la URL pública que consumirá el navegador. Si todo está detrás de este proxy, normalmente será `https://consulecei.com` o una ruta `https://consulecei.com/api` según tu backend.
- `ENV_NAME`, `APP_VERSION`: opcionales (solo informativos para `env.js`).

## Paso 2: Emitir el certificado de Let's Encrypt (método webroot, sin downtime)
Usaremos el webroot que el proxy sirve desde `./certbot-webroot`.

1) Crea el directorio de webroot (si no existe):
```
mkdir -p certbot-webroot
```

2) Asegúrate de que el reverse proxy aún no ocupa el puerto 80 (si ya lo levantaste, puedes bajarlo temporalmente):
```
docker compose down
```

3) Ejecuta certbot con método webroot en el HOST (no en contenedor), apuntando al path absoluto de `certbot-webroot` en tu servidor:
```
# Ajusta RUTA_ABSOLUTA al path real de este repo en tu servidor
sudo certbot certonly --webroot -w RUTA_ABSOLUTA/certbot-webroot -d consulecei.com \
  --email tu-email@dominio.com --agree-tos -n
```
Esto generará los certificados en `/etc/letsencrypt/live/consulecei.com/` (host).

Renovación automática: Certbot ya configurará la renovación. Como usamos webroot, no hay downtime. Sólo necesitarás recargar Nginx tras renovar:
```
sudo certbot renew --dry-run
# Luego, en cada renovación real (cron de certbot):
docker compose exec reverse-proxy nginx -s reload || true
```

> Alternativa con downtime: `certbot certonly --standalone -d consulecei.com` (parando temporalmente el proxy para liberar el puerto 80). No recomendado si puedes usar webroot.

## Paso 3: Levantar el stack
Desde la raíz `consule/`:
```
docker compose up -d --build
```
Abre en el navegador: `https://consulecei.com`

## Variables de entorno del frontend
El `frontend` soporta estas variables en tiempo de ejecución (se procesan desde `env.template.js` por el script de entrypoint):
- `API_BASE_URL` (importante)
- `ENV_NAME` (opcional)
- `APP_VERSION` (opcional)
- `FEATURE_FLAGS` (opcional)

## Jenkins (opcional)
El `Jenkinsfile` está en `frontend/` y permite:
- Build de la app y de la imagen Docker.
- Tests headless (opcional).
- Push a un registry si configuras credenciales.
- Despliegue con `docker compose` en un agente con Docker.

Parámetros clave del `Jenkinsfile` de frontend:
- `DOCKER_IMAGE`: nombre de la imagen (ej. `tuusuario/consule-frontend`).
- `PUSH_IMAGE`: publicar imagen (solo en `main`).
- `DOCKER_REGISTRY_URL`, `DOCKER_CREDENTIALS_ID`.
- `DEPLOY`: si está en `true`, ejecuta `docker compose up -d --build`.

## ¿Y el backend?
En `backend/` no se detectó un `Dockerfile`. Si deseas orquestarlo también con Compose:
1. Añade un `Dockerfile` en `backend/` (o usa una imagen existente).
2. Descomenta y ajusta el servicio `backend` de ejemplo dentro de `docker-compose.yml`.
3. Asegúrate de que `API_BASE_URL` apunte a la URL pública donde el navegador pueda alcanzar tu backend (idealmente bajo `https://consulecei.com`).

## Seguridad y buenas prácticas
- HSTS está activado en el proxy (preload). Asegúrate de que todo tu sitio funciona en HTTPS antes de ponerlo en producción.
- Evita “mixed content”: si sirves por `https://consulecei.com`, configura `API_BASE_URL` también con `https://…`.
- Los certificados se montan como solo lectura en el contenedor (`/etc/letsencrypt`). Las renovaciones las gestiona certbot en el host.
- El frontend no expone puertos públicos; solo el proxy publica 80/443.

## Troubleshooting
- Certbot no valida el reto: verifica que `http://consulecei.com/.well-known/acme-challenge/test` se sirva desde `certbot-webroot`. Revisa DNS/Firewall.
- Error SSL en Nginx: asegúrate de que existen `fullchain.pem` y `privkey.pem` en `/etc/letsencrypt/live/consulecei.com/`.
- Cambié el dominio: actualiza `.env`, emite nuevos certs y edita `reverse-proxy/nginx.conf` (server_name y rutas de certificados).
- Variables no aplican: verifica `/usr/share/nginx/html/env.js` dentro del contenedor (`docker exec -it consule-frontend sh -c 'cat /usr/share/nginx/html/env.js'`).
