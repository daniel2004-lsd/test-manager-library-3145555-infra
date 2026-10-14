# test-manager-library-3145555-infra

> **Prueba técnica · Ficha ADSO 3145555**
> Horario: **miércoles 14 de octubre de 2026, de 6:50 a. m. a 11:00 a. m.** (hora Colombia). Entrega a las **11:00 a. m.**

Este repositorio es la **infraestructura** de *Manager Library*: contenedores, red, volúmenes y el motor de base de datos vacío. **Empieza vacío a propósito**: se construye en el fork de cada aprendiz.

## Instrucciones

Cada aprendiz debe **crear el fork** de los seis repositorios del proyecto y **resolver el proyecto
con el spec planteado**.

1. Hacer fork, a su cuenta de GitHub, de cada repositorio de la tabla del final.
2. Leer el spec en [`test-manager-library-3145555-docs`](https://github.com/code-sena/test-manager-library-3145555-docs).
   Se entrega en dos versiones: `spec-python/` y `spec-.net/`.
3. Desarrollar en los forks.

## El reto se desarrolla con React y PHP (Laravel)

El spec está escrito para Python y para .NET, pero el reto **no** se hace en esos lenguajes:

| Capa | Tecnología del reto |
|---|---|
| Frontend | React |
| Backend | PHP con Laravel |

Lo que el spec define sobre el negocio —historias, criterios de aceptación, reglas, contrato de la
API, modelo de datos— se respeta. Lo que define sobre la tecnología se traduce a React y Laravel.

## La prueba no consiste en escribir el código

El propósito principal es ver la **capacidad de desempeño con SDD** (*Spec-Driven Development*,
desarrollo guiado por especificación): cómo se lee, se interpreta y se aplica una especificación
para llevarla a un stack distinto. El código es el medio, no el fin.

## Entrega

Cuenta lo que quede **subido a los forks antes de las 11:00 a. m.** Lo que llegue después no se revisa.
No hay prórroga: suban avances parciales con commits pequeños durante la prueba.

## Los seis repositorios

| Repositorio | Qué va ahí |
|---|---|
| [`test-manager-library-3145555-docs`](https://github.com/code-sena/test-manager-library-3145555-docs) | El spec: `spec-python/` y `spec-.net/` |
| [`test-manager-library-3145555-api`](https://github.com/code-sena/test-manager-library-3145555-api) | Backend en PHP (Laravel) |
| [`test-manager-library-3145555-app`](https://github.com/code-sena/test-manager-library-3145555-app) | Frontend en React |
| [`test-manager-library-3145555-page`](https://github.com/code-sena/test-manager-library-3145555-page) | Sitio público estático de presentación |
| [`test-manager-library-3145555-infra`](https://github.com/code-sena/test-manager-library-3145555-infra) | Contenedores, red, volúmenes y motor de base de datos vacío |
| [`test-manager-library-3145555-tool`](https://github.com/code-sena/test-manager-library-3145555-tool) | Utilidades: sembrador de datos de demostración |

---

# Infraestructura (este repositorio)

Todo el proyecto levanta con un solo comando (RNF-5, T-20):

```bash
docker compose up -d
```

Requisitos: Docker con el plugin Compose v2, y los seis repos clonados como **directorios hermanos**:

```
manager-library-3145555/
├── test-manager-library-3145555-api      # contexto de build del servicio api
├── test-manager-library-3145555-app      # contexto de build del servicio app
├── test-manager-library-3145555-page     # sitio estático servido por el servicio page
├── test-manager-library-3145555-docs
├── test-manager-library-3145555-infra    # ESTE repositorio
└── test-manager-library-3145555-tool
```

## Servicios

| Servicio | Imagen / build | Puerto (host→contenedor) | Qué hace |
|---|---|---|---|
| `db` | `postgres:16-alpine` | `5433→5432` | Motor PostgreSQL 16, servidor UTC, **base vacía** |
| `api` | build de `../test-manager-library-3145555-api` | `8000→8000` | Laravel (`php artisan serve`) |
| `app` | build de `../test-manager-library-3145555-app` | `5173→80` | Frontend React |
| `page` | `nginx:alpine` | `8081→80` | Sitio público estático (no habla con la API) |

Red nombrada `library-net` (la `db` solo es accesible desde ahí; el 5433 se publica únicamente para depurar con `psql`). Volúmenes nombrados `library-db-data` (datos) y `library-covers` (portadas subidas a la API).

## La base arranca VACÍA a propósito (ADR-001)

Este repo **no contiene `CREATE TABLE` ni ningún script SQL init**: el esquema de la aplicación
(esquema `library`, tablas, índices, restricciones y semilla) es **propiedad de las migraciones
del repo `api`**. Ver [ADR-001](https://github.com/code-sena/test-manager-library-3145555-docs) —
Artículo V de la constitución.

Verificar que la base está vacía (sin tablas de la aplicación):

```bash
docker compose exec db psql -U library -d library -c '\dt'
docker compose exec db psql -U library -d library -c '\dn'
```

`\dt` debe devolver `(0 filas)`. El único esquema presente es el `public` por defecto; el esquema `library` lo crea la migración del `api`.

## Migrar después de levantar (la base está vacía; el esquema vive en el api)

```bash
docker compose exec api php artisan migrate --force
```

## Si algo falla

- **El servicio `app` necesita que el repo `../test-manager-library-3145555-app` ya tenga `Dockerfile`.**
  Mientras el frontend no esté listo, levanta lo esencial: `docker compose up -d db api page`
  (el servicio `page` solo monta el directorio estático y no requiere build).
- **La API (`artisan serve`) necesita su código y `vendor/`** en el contexto de build
  `../test-manager-library-3145555-api`. Si el backend aún no arranca, revisa:
  `docker compose logs api`.
- **Desarrollo del frontend sin contenedor**: en el repo `app`, `npm run dev` apuntando a la API con
  `VITE_API_URL=http://localhost:8000` (el backend acepta CORS en `http://localhost:5173`).

## Secretos y variables

Copia `.env.example` a `.env` (el `.env` real está en `.gitignore`; Artículo IX). Variables:
`POSTGRES_PASSWORD`, `JWT_SECRET`, `ADMIN_PASSWORD`, `APP_KEY` y los puertos
`DB_HOST_PORT`, `API_PORT`, `APP_PORT`, `PAGE_PORT`, `APP_CONTAINER_PORT`.
Los defaults sirven para desarrollo local.

## Comandos útiles

```bash
docker compose up -d                     # todo
docker compose up -d db api              # solo motor + API (lo esencial)
docker compose logs -f api               # logs de la API
docker compose exec db psql -U library -d library -c '\dt'   # base vacía
docker compose exec api php artisan migrate --force          # crear el esquema
docker compose down                       # parar (conserva datos)
docker compose down -v                    # parar y borrar el volumen (base limpia)
```
