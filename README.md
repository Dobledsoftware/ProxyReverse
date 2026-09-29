# ProxyReverse — Nginx Reverse Proxy

Contenedor nginx que actúa como reverse proxy para todos los servicios de Dobledsoftware en el servidor de producción/test.

## Estructura

```
ProxyReverse/
├── docker-compose.yml
└── nginx/
    └── default.conf     # configuración de todos los virtual hosts
```

En el servidor, este repo vive en `/var/apps/reverseProxy`. El contenedor `nginx_proxy` además monta `/var/apps/landings` (ver sección "Landings estáticas" más abajo).

## Cómo funciona

Nginx escucha en el puerto 80 y redirige cada request al container correspondiente según el `server_name` (dominio). Todos los containers comparten la red externa `reverseproxy_proxy-net`, lo que permite que nginx los resuelva por nombre. Las landings estáticas son la excepción: no tienen container propio, nginx las sirve directo desde disco (`root`).

### Dominios configurados

| Dominio | Frontend | Backend |
|---|---|---|
| `dobledsoftware.com.ar` | `dobledsoftware-frontend:5172` | — |
| `odoo.dobledsoftware.com.ar` | `odoo_dev:8069` | — |
| `stock-dev.dobledsoftware.com.ar` | `stock_front:80` | `stock_api:8001` |
| `stock-qa.dobledsoftware.com.ar` | `stock_front_qa:80` | `stock_api_qa:8001` |
| `stock-uat.dobledsoftware.com.ar` | `stock_front_uat:80` | `stock_api_uat:8001` |
| `auth.dobledsoftware.com.ar` | `keycloak_dev:8080` | — |
| `facturastest.dobledsoftware.com.ar` | `facturas_front:5181` | `facturas_api:8000` |
| `corralontest.dobledsoftware.com.ar` | `stock_front_corralon_test:5182` | `stock_api_corralon_test:8001` |
| `agendastest.dobledsoftware.com.ar` | `agenda-front:80` | `agenda-srv:8091` |
| `dra-davila.dobledsoftware.com.ar`, `abogadaluciadavila.com.ar` | landing estática (`/var/apps/landings/dra-davila`) | — |
| `barberiavip.dobledsoftware.com.ar`, `clinicasol.dobledsoftware.com.ar` | redirect 301 a `agendastest` | — |

`stock-dev`/`stock-qa`/`stock-uat` reemplazan a `legendary`/`legendarytest` (dados de baja) — un dominio por ambiente de la cadena `develop → QA → UAT → main` de stock-srv/stock-front (ver `AGENTS.md` de esos repos, issue #70/#49).

## Resolver DNS dinámico (importante)

### El problema

Nginx resuelve los nombres de los upstreams (containers) **en el momento de arrancar**. Si cualquier container referenciado en el config **no está corriendo** en ese momento, nginx falla al iniciar y **todos los sitios caen**, incluyendo los que sí están activos.

### La solución

Cada `location` usa el resolver DNS interno de Docker (`127.0.0.11`) junto con una variable para el `proxy_pass`:

```nginx
location / {
    resolver 127.0.0.11 valid=30s;
    set $upstream http://nombre-container:puerto;
    proxy_pass $upstream;
}
```

Con este patrón, nginx resuelve el nombre del container **en cada request** (con caché de 30 segundos), no al arrancar. Si un container está caído, solo ese dominio devuelve error — el resto sigue funcionando normalmente.

### Rutas de API con strip de prefijo

Los backends que no incluyen `/api/` en sus rutas usan `rewrite` para eliminar el prefijo antes de hacer el proxy:

```nginx
location /api/ {
    resolver 127.0.0.11 valid=30s;
    set $upstream http://backend:8001;
    rewrite ^/api(/.*)$ $1 break;
    proxy_pass $upstream;
}
```

Esto transforma `/api/items` → `/items` antes de llegar al backend.

Los backends que sí incluyen `/api/` en sus rutas (como `agenda-srv`) no necesitan el rewrite:

```nginx
location /api/ {
    resolver 127.0.0.11 valid=30s;
    set $upstream http://agenda-srv:8091;
    proxy_pass $upstream;   # pasa /api/auth/login tal cual
}
```

## Landings estáticas (consolidadas)

Las landings de clientes (sitios estáticos, sin backend propio) **no** tienen un contenedor `nginx:alpine` dedicado cada una — a partir de la consolidación de septiembre 2026 se sirven todas directo desde este mismo `nginx_proxy`, leyendo archivos de `/var/apps/landings/<slug>/` (montado read-only en el container, ver `docker-compose.yml`). Motivo: con 1 contenedor por cliente, 20+ landings futuras implican 20+ `docker run` manuales, un `default.conf` cada vez más largo de mantener y ningún ahorro real de recursos (el costo marginal de un container de más es ~1KB + los archivos del sitio, la imagen base ya está compartida).

### Sumar una landing nueva

1. Subí los archivos estáticos a `/var/apps/landings/<slug>/` en el servidor (`index.html`, `assets/`, etc. — mismo mecanismo que se usaba antes para el bind mount del contenedor dedicado, por ejemplo `pscp`).
2. Copiá este bloque en `nginx/default.conf`, cambiando `server_name` y el path de `root`:
   ```nginx
   server {
       listen 80;
       server_name nueva-landing.dobledsoftware.com.ar;

       root /var/apps/landings/<slug>;
       index index.html;

       location / {
           try_files $uri $uri/ =404;
       }
   }
   ```
3. Si la landing necesita una URL de preview para aprobar cambios antes de producción (ej. `/preview/`), basta con que esos archivos vivan en un subdirectorio de `<slug>/` — no hace falta un `location` ni un `alias` aparte, `try_files` ya lo resuelve porque cuelga del mismo `root`.
4. Seguí el flujo de despliegue de más abajo (rama `release-*`, o aplicación manual si es un cambio de emergencia).

## Agregar un nuevo servicio

1. Agregá el container a la red `reverseproxy_proxy-net` en su propio `docker-compose.yml`:
   ```yaml
   networks:
     reverseproxy_proxy-net:
       external: true
   ```

2. Agregá un bloque `server` en `nginx/default.conf` siguiendo el patrón con resolver:
   ```nginx
   server {
       listen 80;
       server_name nuevo-servicio.dobledsoftware.com.ar;

       location / {
           resolver 127.0.0.11 valid=30s;
           set $upstream http://nombre-container:puerto;
           proxy_pass $upstream;
           proxy_set_header Host $host;
           proxy_set_header X-Real-IP $remote_addr;
           proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
       }
   }
   ```

3. Commiteá y pusheá — el deploy se aplica solo (ver "Despliegue" abajo).

## Despliegue

El deploy es automático vía GitHub Actions (`.github/workflows/deploy.yml`): pushear a cualquier rama `release-*` dispara el workflow, que se conecta por SSH al servidor, hace `git reset --hard`/`git clean -fd` sobre `/var/apps/reverseProxy` (o sea: lo que esté commiteado en esa rama pasa a ser la verdad — no dejar cambios sueltos hechos a mano en el servidor sin commitear), valida la sintaxis con un contenedor `nginx:latest` descartable, y si es válida recrea `nginx_proxy` (`docker rm -f` + `docker run`, no `nginx -s reload` — un reload no ve un `default.conf` reemplazado por bind-mount de un solo archivo, mantiene el inodo viejo).

Notifica por mail (`ADMIN_EMAIL`) si el deploy salió bien o mal.

### Aplicación manual (sin pasar por `release-*`)

Para un cambio urgente que no amerita esperar un release, el mismo patrón a mano en el servidor:

```bash
# Validar ANTES de tocar el contenedor en vivo
docker run --rm -v /var/apps/reverseProxy/nginx/default.conf:/etc/nginx/conf.d/default.conf:ro -v /var/apps/landings:/var/apps/landings:ro nginx:latest nginx -t

# Recrear (no reload)
docker rm -f nginx_proxy
docker run -d --name nginx_proxy -p 80:80 \
  -v /var/apps/reverseProxy/nginx/default.conf:/etc/nginx/conf.d/default.conf \
  -v /var/apps/landings:/var/apps/landings:ro \
  --network reverseproxy_proxy-net --restart always nginx:latest
```

Importante: si se aplica así, el cambio queda **solo en el servidor** hasta que alguien lo commitee — el próximo deploy a `release-*` lo pisa con `git reset --hard`. Commitear el cambio real (incluso después de aplicarlo a mano) es responsabilidad de quien lo aplicó.
