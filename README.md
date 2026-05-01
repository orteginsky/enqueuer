# Enqueuer Universal — Cloud Tasks Dispatcher

Servicio **agnóstico y reutilizable** que actúa como Gateway para crear tareas en Google Cloud Tasks.

Este servicio permite desacoplar tus clientes (Frontends, Webhooks, Scripts) de tus Workers (Cloud Run), gestionando automáticamente:
- **Autenticación OIDC** (Service-to-Service).
- **Forwarding de API Keys** (Client-to-Worker).
- **Configuración Dinámica** (Hot-reload desde Cloud Storage sin redesplegar).

---

## ✨ Características

- **Configuración Externa:** Lee la definición de colas y URLs desde un archivo `services.json` alojado en un Bucket de GCS.
- **Cache Inteligente:** Mantiene la configuración en memoria (TTL 5 min) y se actualiza sola.
- **Auth Forwarding:** Si el cliente envía una `worker_api_key`, el Enqueuer la inyecta como header `X-API-Key` al llamar al worker.
- **Resilient:** Maneja reintentos, timeouts extendidos (hasta 30 min) y dead-letter queues vía Cloud Tasks.

---

## 🧱 Arquitectura

```text
                                    ┌── [ GCS Bucket: services.json ]
                                    │      (Configuración dinámica)
                                    ▼
Client ──(POST /enqueue)──> [ Enqueuer API ] 
                            (Cloud Run)
                                    │
                                    ▼
                            [ Cloud Tasks ] 
                            (Queue: queue-filtros | queue-brain...)
                                    │
                                    │ (HTTP POST + OIDC Token + X-API-Key)
                                    ▼
                            [ Worker Service ]
                            (Cloud Run: filtros-ai | brain...)

```

---

## 🗂 Estructura del Proyecto

```text
enqueuer/
├─ app.py                # Lógica principal (FastAPI + GCS Loader)
├─ config.py             # (Opcional) Gestor de configuración
├─ services.json         # Plantilla local de servicios
├─ deploy_universal.sh   # 🚀 Script de despliegue automático
├─ requirements.txt
├─ Dockerfile
└─ .dockerignore

```

---

## ⚙️ Configuración (services.json)

Ya no necesitas redeplegar el Enqueuer para agregar un worker. Solo edita el archivo `services.json` en tu Bucket (`gs://<PROJECT_ID>-enqueuer-configs/services.json`).

**Formato:**

```json
{
  "filtros-ai": {
    "queue": "queue-filtros-ai",
    "url": "[https://filtros-ai-service-XYZ.run.app/api/v1/process](https://filtros-ai-service-XYZ.run.app/api/v1/process)",
    "aud": "[https://filtros-ai-service-XYZ.run.app](https://filtros-ai-service-XYZ.run.app)",
    "deadline_s": 1500
  },
  "brain": {
    "queue": "queue-brain",
    "url": "[https://brain-service-XYZ.run.app/process](https://brain-service-XYZ.run.app/process)",
    "aud": "[https://brain-service-XYZ.run.app](https://brain-service-XYZ.run.app)",
    "deadline_s": 600
  }
}

```

* `queue`: Nombre de la cola en Cloud Tasks.
* `url`: Endpoint completo del worker.
* `aud`: URL base del servicio (para firmar el token OIDC).
* `deadline_s`: Tiempo máximo de ejecución antes de timeout (máx 1800s para HTTP).

---

## 🚀 Despliegue Universal

Este proyecto incluye un script maestro que configura **todo** (APIs, IAM, Bucket, Docker Build & Deploy) automáticamente en cualquier proyecto de GCP.

### Requisitos previos

* Google Cloud SDK (`gcloud`) instalado y logueado.
* Permisos de Owner o Editor en el proyecto destino.

### Pasos

1. Sitúate en la carpeta del proyecto.
2. Ejecuta el script:

```bash
chmod +x deploy_universal.sh
./deploy_universal.sh

```

El script hará lo siguiente:

1. Habilitará APIs (`cloudtasks`, `run`, `storage`, `cloudbuild`).
2. Creará la Service Account `enqueuer-sa`.
3. Asignará permisos IAM necesarios (Invoker, Enqueuer, TokenCreator, StorageViewer).
4. Creará el Bucket de configuración y subirá tu `services.json` local.
5. Desplegará el servicio en Cloud Run.

---

## 🏃 Desarrollo Local

Para probar en tu máquina sin dockerizar:

1. **Entorno Virtual:**
```bash
python -m venv venv
source venv/bin/activate
pip install -r requirements.txt

```


2. **Credenciales:**
Necesitas una llave JSON de la Service Account creada (`enqueuer-sa`) para tener permiso de lectura al bucket y creación de tareas.
```bash
gcloud iam service-accounts keys create enqueuer-key.json --iam-account=enqueuer-sa@<PROJECT_ID>.iam.gserviceaccount.com

```


3. **Variables (.env):**
```properties
PROJECT_ID=tu-proyecto-id
TASKS_REGION=us-central1
CALLER_SA=enqueuer-sa@tu-proyecto-id.iam.gserviceaccount.com
CONFIG_BUCKET_NAME=tu-proyecto-id-enqueuer-configs
CONFIG_FILE_NAME=services.json
GOOGLE_APPLICATION_CREDENTIALS=./enqueuer-key.json

```


4. **Ejecutar:**
```bash
uvicorn app:app --reload

```



---

## 📮 API Reference

### `POST /enqueue`

Encola una tarea para ser procesada asíncronamente.

**Body:**

```json
{
  "service": "filtros-ai",
  "worker_api_key": "mi-super-secreto-123", 
  "payload": {
    "task_id": "REQ-001",
    "data": "Cualquier JSON que espere tu worker"
  },
  "delay_s": 0,
  "idempotency_key": "unique-uuid-v4"
}

```

* `service`: Clave definida en `services.json`.
* `worker_api_key` *(Opcional)*: Si se envía, el Enqueuer la reenviará al Worker en el header `X-API-Key`.
* `payload`: Datos que recibirá el worker en el body.
* `delay_s`: Segundos a esperar antes de ejecutar.
* `idempotency_key`: ID único para evitar ejecuciones duplicadas.

**Respuesta (200 OK):**

```json
{
  "ok": true,
  "task": "projects/.../tasks/123456",
  "service": "filtros-ai",
  "forwarded_api_key": true
}

```

---

## 🔄 Recarga de Configuración

El servicio recarga la configuración del Bucket automáticamente cada 5 minutos. Si necesitas forzar una actualización inmediata (ej. acabas de subir un cambio al JSON):

**Request:**
`POST /config/refresh`

---

## 🧰 Troubleshooting común

* **Error 403 (Storage):** La SA no tiene permiso para leer el bucket. Ejecuta:
```bash
gcloud storage buckets add-iam-policy-binding gs://<BUCKET> --member="serviceAccount:<SA>" --role="roles/storage.objectViewer"

```


* **Worker devuelve 401/403:**
* Verifica que estás enviando `worker_api_key` en el payload de `/enqueue`.
* Verifica que la SA del Enqueuer tenga `roles/run.invoker` en el proyecto.


* **Task falla con DEADLINE_EXCEEDED:**
* El proceso dura más que la configuración. Aumenta `deadline_s` en el JSON y `timeoutSeconds` en el Cloud Run del worker.



---

## 🧾 Licencia

MIT
