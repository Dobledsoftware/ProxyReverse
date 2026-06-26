# ProxyReverse — Nginx Reverse Proxy

Contenedor nginx que actúa como reverse proxy para todos los servicios de Dobledsoftware en el servidor de producción/test.

## Estructura

```
ProxyReverse/
├── docker-compose.yml
└── nginx/
    └── default.conf     # configuración de todos los virtual hosts
```

## Cómo funciona

Nginx escucha en el puerto 80 y redirige cada request al container correspondiente según el `server_name` (dominio). Todos los containers comparten la red externa `reverseproxy_proxy-net`, lo que permite que nginx los resuelva por nombre.

### Dominios configurados

| Dominio | Frontend | Backend |
|---|---|---|
| `dobledsoftware.com.ar` | `dobledsoftware-frontend:5172` | — |
| `odoo.dobledsoftware.com.ar` | `odoo_dev:8069` | — |
| `legendary.dobledsoftware.com.ar` | `stock_front:5174` | `stock_api:8001` |
| `auth.dobledsoftware.com.ar` | `keycloak_dev:8080` | — |
| `facturastest.dobledsoftware.com.ar` | `facturas_front:5181` | `facturas_api:8000` |
| `legendarytest.dobledsoftware.com.ar` | `stock_front_test:5173` | `stock_api_test:8001` |
| `corralontest.dobledsoftware.com.ar` | `stock_front_corralon_test:5182` | `stock_api_corralon_test:8001` |
| `agendastest.dobledsoftware.com.ar` | `agenda-front:80` | `agenda-srv:8091` |

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

3. Commiteá, pusheá, y en el servidor:
   ```bash
   cd /var/apps/reverseProxy
   git pull
   docker-compose down && docker-compose up -d
   ```

## Despliegue

```bash
# En el servidor
cd /var/apps/reverseProxy
git pull
docker-compose down && docker-compose up -d

# Verificar que arrancó sin errores
docker logs nginx_proxy --tail 20
```
