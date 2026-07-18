# kyros-deploy

Orquesta el despliegue de Kayros (backend Spring Boot + frontend Next.js + MySQL + Caddy) con
Docker Compose. No contiene código de la aplicación — solo la infraestructura. El código vive en
sus propios repos, como carpetas hermanas de esta:

```
KyrosSystem/
├── Back/demokyros/       (repo del backend)
├── Front/kyros-front/    (repo del frontend)
└── kyros-deploy/         (este repo)
```

## Requisitos

- Docker + Docker Compose (`docker compose version`).
- Los repos `Back/demokyros` y `Front/kyros-front` clonados como carpetas hermanas de este repo
  (mismo nivel, ver árbol arriba).

## Probar todo el stack en local (sin dominio ni servidor)

```bash
cp .env.example .env
# Deja DOMAIN=:80 tal cual viene en el ejemplo — sirve HTTP plano en localhost.
docker compose up --build
```

Abre `http://localhost` — debe cargar el login. `docker compose logs -f <servicio>` para ver logs
de `mysql`, `backend`, `frontend` o `caddy` por separado.

Para parar y borrar todo (incluyendo los datos de MySQL y las fotos subidas):
`docker compose down -v`. Sin el `-v`, `docker compose down` conserva los volúmenes.

## Desplegar en un servidor (Hetzner u otro)

1. Servidor Ubuntu con Docker + Docker Compose instalados, con solo los puertos 22/80/443
   abiertos en el firewall.
2. Dominio con un registro A apuntando a la IP del servidor.
3. En el servidor, clonar los 3 repos como hermanos (mismo árbol de arriba).
4. `cp .env.example .env` y llenar valores reales:
   - `DOMAIN=tudominio.com`
   - `DB_PASSWORD` / `MYSQL_ROOT_PASSWORD`: contraseñas fuertes, distintas entre sí.
   - `JWT_SECRET`: `openssl rand -base64 32`.
   - `CORS_ALLOWED_ORIGINS=https://tudominio.com`
5. `docker compose up -d --build`. Caddy provisiona el certificado HTTPS solo, la primera vez
   puede tardar unos segundos en emitirlo (Let's Encrypt).
6. Verificar: `docker compose ps` (todo en estado `healthy`/`running`), `docker compose logs -f`.

## Actualizar tras un cambio de código

```bash
cd ../Back/demokyros && git pull
cd ../../Front/kyros-front && git pull
cd ../kyros-deploy
docker compose up -d --build
```

## Qué NO se expone a internet

Solo Caddy publica puertos al host (80/443). `mysql` y `backend` solo son alcanzables dentro de
la red interna de Docker Compose — ni la base de datos ni la API quedan accesibles directo desde
internet, todo pasa por Caddy.

## Persistencia

- `mysql-data`: la base de datos.
- `uploads-data`: fotos de alumnos (el backend las guarda en disco, no en la BD — ver
  `app.uploads.dir` en `application.properties`).
- `caddy-data`: certificados HTTPS, para no volver a pedirlos a Let's Encrypt en cada reinicio.

Ninguno de estos tres se borra con `docker compose down` (sin `-v`) ni con `up --build`. Hacer
backup periódico de `mysql-data` (`mysqldump`) y `uploads-data` sigue siendo necesario — no hay
nada de esto respaldado por git.
