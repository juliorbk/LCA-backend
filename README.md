# omnichannel-core

Backend del **CMS Omnicanal** — publica contenido en Telegram, Instagram/Facebook y WhatsApp desde un único endpoint, de forma paralela y asíncrona.

Desarrollado como proyecto de Servicio Comunitario · UNEG · v1.0

---

## Stack tecnológico

| Capa | Tecnología |
|---|---|
| API | Python 3.12 + FastAPI |
| Asincronía | `asyncio` + `asyncio.gather()` |
| ORM | SQLAlchemy 2.0 async |
| Migraciones | Alembic |
| Base de datos | PostgreSQL 16 |
| Seguridad | python-jose (JWT) + passlib (bcrypt) + cryptography (Fernet) |
| Storage | boto3 (S3) / Cloudinary |
| HTTP client | httpx async |
| Task queue | Celery + Redis |
| Contenedores | Docker + docker-compose |

---

## Estructura del proyecto

```
omnichannel-core/
├── app/
│   ├── main.py                        # Instancia FastAPI, registra routers
│   ├── core/
│   │   ├── config.py                  # Settings con Pydantic BaseSettings
│   │   ├── database.py                # Engine async, dependencia get_db
│   │   └── security.py               # JWT, bcrypt, Fernet encrypt/decrypt
│   ├── models/
│   │   ├── user.py
│   │   ├── post.py
│   │   ├── social_credential.py      # Tokens cifrados con Fernet
│   │   └── publication_log.py        # Estados: PENDING | SENT | ERROR
│   ├── schemas/
│   │   ├── auth.py
│   │   ├── publish.py
│   │   └── credential.py
│   ├── services/
│   │   ├── orchestrator.py           # asyncio.gather() para los 3 canales
│   │   ├── telegram_service.py
│   │   ├── meta_service.py           # upload_container → publish_post
│   │   ├── whatsapp_proxy.py         # Proxy HTTP al microservicio Node.js
│   │   └── media_service.py          # Upload a S3 o Cloudinary
│   ├── api/v1/
│   │   ├── deps.py                   # CurrentUser, DB
│   │   └── routes/
│   │       ├── auth.py               # POST /api/v1/auth/login|register
│   │       ├── publish.py            # POST /api/v1/publish
│   │       └── credentials.py       # CRUD /api/v1/credentials
│   └── tasks/
│       ├── celery_app.py
│       └── publish_tasks.py          # Tarea retry_publish
├── alembic/
│   └── versions/
├── tests/
├── docker-compose.yml
├── requirements.txt
└── .env.example
```

---

## Endpoints principales

| Método | Ruta | Descripción |
|---|---|---|
| `POST` | `/api/v1/auth/register` | Registro de usuario |
| `POST` | `/api/v1/auth/login` | Login → JWT |
| `POST` | `/api/v1/publish` | Publica en los 3 canales en paralelo |
| `GET` | `/api/v1/credentials` | Lista tokens guardados |
| `POST` | `/api/v1/credentials` | Guarda token de red social |
| `DELETE` | `/api/v1/credentials/{id}` | Elimina token |
| `GET` | `/health` | Health check |

---

## Patrón de orquestación

Los tres canales se publican en paralelo con `asyncio.gather()`. Si uno falla, los demás continúan y el log registra el estado individual de cada plataforma.

```python
results = await asyncio.gather(
    telegram_service.send(post),
    meta_service.publish(post),
    whatsapp_proxy.send(post),
    return_exceptions=True
)
```

---

## Instalación local

### Requisitos
- Python 3.12
- PostgreSQL 16
- Redis
- Docker (opcional)

### Con Docker

```bash
cp .env.example .env
# completar variables en .env
docker-compose up --build
```

### Sin Docker

```bash
python -m venv venv
source venv/bin/activate        # Windows: venv\Scripts\activate
pip install -r requirements.txt

# aplicar migraciones
alembic upgrade head

# iniciar servidor
uvicorn app.main:app --reload
```

La documentación interactiva estará disponible en `http://localhost:8000/docs`.

---

## Variables de entorno

Copiar `.env.example` a `.env` y completar:

```env
DATABASE_URL=postgresql+asyncpg://usuario:password@localhost:5432/omnichannel
REDIS_URL=redis://localhost:6379/0
SECRET_KEY=cambiar-en-produccion
ENCRYPTION_KEY=clave-fernet-base64-32-bytes

# Telegram
TELEGRAM_BOT_TOKEN=

# Meta
META_APP_ID=
META_APP_SECRET=

# WhatsApp microservice
WA_SERVICE_URL=http://localhost:3001
CORE_SECRET=shared-secret

# Storage (s3 o cloudinary)
STORAGE_BACKEND=s3
AWS_ACCESS_KEY_ID=
AWS_SECRET_ACCESS_KEY=
AWS_S3_BUCKET=
```

---

## Fases de desarrollo

| Fase | Nombre | Entregables |
|---|---|---|
| **Fase 1** | Quick Win — Telegram | PostgreSQL + Alembic · FastAPI base · TelegramService · PublicationLog |
| **Fase 2** | Núcleo Meta (FB + Instagram) | OAuth 2.0 · MetaService · MediaService · S3 |
| **Fase 3** | Experimento WhatsApp | Microservicio Node.js · Baileys · whatsapp_proxy.py |

---

## Notas importantes

> **RIESGO — WhatsApp:** Meta puede banear números que usen automatización no oficial (Baileys). Usar siempre un número de prueba dedicado.

> **Meta Graph API** requiere cuentas Business/Professional de Instagram vinculadas a una Página de Facebook. Verificar antes de iniciar la Fase 2.

---

## Microservicio relacionado

El directorio `whatsapp-service/` contiene el microservicio Node.js + TypeScript con Baileys que maneja la sesión de WhatsApp. Se comunica con este backend mediante un shared secret (`CORE_SECRET`).
