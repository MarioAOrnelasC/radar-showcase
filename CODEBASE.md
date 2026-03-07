# CODEBASE.md — RADAR
## Mapa de Arquitectura del Código

Este documento describe la estructura del backend de RADAR, el propósito de cada archivo clave, su importancia relativa y qué partes son más relevantes para entender el sistema. El frontend es una app React de página única (`App.jsx`) cuyo build se sirve como archivos estáticos desde Flask.

---

## Visión General de la Arquitectura

```
backend/
├── app/
│   ├── main.py                     ← Punto de entrada FastAPI + routers
│   ├── config.py                   ← Configuración centralizada (env vars)
│   ├── hybrid_pipeline/            ← Pipeline principal de scraping+ML
│   │   ├── pipeline.py             ← Orquestador del pipeline
│   │   └── stages/                 ← Etapas individuales del pipeline
│   ├── scrapers/                   ← Scrapers por fuente y dominio
│   │   ├── federal/                ← SAM.gov, Grants.gov
│   │   ├── state/                  ← Illinois BidBuy
│   │   ├── municipal/              ← Chicago, Cook County
│   │   ├── diversity/              ← Supplier diversity programs
│   │   ├── saas/                   ← OpenGov, Playwright, NoDriver
│   │   └── corporate/              ← Jaggaer, Workday
│   ├── database/
│   │   └── qdrant_client.py        ← Cliente Qdrant (Singleton)
│   ├── models/                     ← Modelos Pydantic (contratos, jobs, config)
│   ├── routers/                    ← Rutas FastAPI por dominio
│   ├── tasks/
│   │   └── job_tasks.py            ← Tareas Celery (worker)
│   ├── nlp/                        ← Motor NLP (spaCy + scikit-learn)
│   ├── utils/                      ← Utilidades compartidas
│   └── scripts/legacy/             ← Scripts obsoletos (referencia)
├── models/                         ← Artefactos ML (.pkl)
├── tests/                          ← Suite pytest
├── pyproject.toml / requirements.txt
└── jobs/                           ← Outputs de jobs (CSV, JSON)
```

---

## Punto de Entrada

### `app/main.py` — ⭐⭐⭐⭐⭐ Crítico
El punto de entrada de la aplicación FastAPI. Registra todos los routers, configura CORS, middleware de sesión, y los eventos de ciclo de vida (`startup`/`shutdown`).

**Función `lifespan()`:** Se ejecuta al arrancar el servidor. Carga los modelos ML (`.pkl`) en `app.state.ml_models` una sola vez, evitando recargarlos en cada request.

```python
# Patrón Singleton de modelos ML en lifespan:
@asynccontextmanager
async def lifespan(app: FastAPI):
    app.state.ml_models = load_ml_models()  # Se ejecuta UNA sola vez
    yield
```

También incluye el endpoint `/healthz` para Render health checks.

---

### `app/config.py` — ⭐⭐⭐⭐⭐ Crítico
Configuración centralizada usando `pydantic-settings`. Todas las variables de entorno se cargan aquí una vez y se acceden por el resto del sistema vía `get_settings()`.

Variables clave:
- `QDRANT_URL`, `QDRANT_API_KEY` — Conexión a Qdrant Cloud.
- `OPENAI_API_KEY` — Embeddings y extracción LLM.
- `REDIS_URL` — Broker Celery y almacén de jobs/archivos.
- `PARALLEL_LLM_WORKERS`, `PARALLEL_FETCH_WORKERS` — Paralelismo configurable.
- `CONTRACT_EXPIRATION_DAYS`, `GRANT_EXPIRATION_DAYS` — Filtros de vencimiento.
- `NLP_ARTIFACTS_DIR` — Ruta al volumen montado en Render con modelos spaCy.

> 🔑 **Nunca hacer commit de `.env`** — contiene todas las claves secretas.

---

## Pipeline Principal

### `app/hybrid_pipeline/pipeline.py` — ⭐⭐⭐⭐⭐ Crítico
El orquestador del pipeline. Recibe la configuración de un job y ejecuta las etapas en secuencia:

```
Normalize → ETL Router → Fetch → Parse → Validate → Enrich → Store
```

Maneja el estado del job, logging de progreso, y errores por etapa. Es el código que Celery ejecuta cuando toma un job de la queue.

---

### `app/hybrid_pipeline/stages/normalize.py` — ⭐⭐⭐⭐ Muy importante
Primera etapa del pipeline. Recibe las URLs del CSV de input y:
1. Canoniza las URLs (elimina parámetros de tracking, normaliza protocolos).
2. Verifica `robots.txt` para cada dominio.
3. Clasifica cada URL en el tipo de scraper correcto (vía el ETL Router).

---

### `app/hybrid_pipeline/stages/parse.py` — ⭐⭐⭐⭐⭐ Crítico
La etapa de mayor impacto. Extrae datos estructurados de contratos desde HTML crudo.

**Estrategia híbrida:**
1. **Parseo determinístico primero:** Reglas específicas por fuente (selectores CSS, patrones regex). Rápido, sin costo de API.
2. **LLM como fallback:** Si el parseo determinístico no extrae los campos requeridos, envía el HTML (truncado) a GPT-4 con un prompt específico.

Ejecuta **5 workers paralelos** (`PARALLEL_LLM_WORKERS`) para maximizar throughput en batches grandes.

---

### `app/hybrid_pipeline/stages/validate.py` — ⭐⭐⭐⭐ Muy importante
Valida y normaliza cada contrato extraído:
- Verifica campos requeridos (título, URL, fecha de cierre).
- Filtra contratos vencidos (configurable via `CONTRACT_EXPIRATION_DAYS`).
- Genera el **`stable_id`** único por contrato (hash de URL canonizada + número de contrato). Este ID es el que se usa para deduplicación en Qdrant.
- Clasifica nivel de gobierno (federal/state/local) y tipo de oportunidad (solicitation/grant/award).

> ⚠️ La función `_generate_stable_id()` aquí es la fuente de verdad. No usar `QdrantDatabase._generate_stable_id()` directamente (genera IDs incompatibles).

---

### `app/hybrid_pipeline/stages/enrich.py` — ⭐⭐⭐⭐ Muy importante
Enriquece cada contrato validado con:
- **Embeddings** (OpenAI `text-embedding-ada-002`, 1536 dimensiones) para búsqueda semántica.
- **Resumen** (GPT-4, modo "full" solamente) para el campo `summary`.
- **Predicción de categoría y NAICS** via modelos ML locales (scikit-learn TF-IDF).

---

### `app/hybrid_pipeline/stages/store.py` — ⭐⭐⭐⭐ Muy importante
Última etapa. Escribe los contratos a:
1. **CSV output** — Contratos y grants en archivos separados. Manejo de errores **por fila** (un registro malo no aborta el batch).
2. **Qdrant Cloud** — En el modo configurado (`incremental`/`upsert`/`replace_all`).
3. **Redis** — Los archivos CSV se serializan en base64 y se guardan en Redis (TTL 24h) para que el servicio web los pueda descargar.

---

## Scrapers

### `app/scrapers/federal/sam_gov_scraper.py` — ⭐⭐⭐⭐⭐ Crítico
**60.8 KB** — El scraper más grande y más crítico. Accede a la API v2 de SAM.gov con paginación de hasta 1,000 resultados por consulta. Incluye:
- Extracción de PDFs adjuntos (documentos de solicitud).
- Clasificación de estado (activo/próximo/vencido).
- Extracción de múltiples códigos NAICS por contrato.
- Fix de timezone para comparación de fechas (`datetime.now(timezone.utc)`).

### `app/scrapers/federal/grants_gov_scraper.py` — ⭐⭐⭐⭐ Muy importante
Scraper de la API de Grants.gov para subvenciones federales. Extrae códigos CFDA, categorías de programa, y requisitos de elegibilidad.

### `app/scrapers/state/illinois_bidbuy_scraper.py` — ⭐⭐⭐ Importante
Scraper del portal de adquisiciones de Illinois (BidBuy). Accede a licitaciones estatales de Illinois.

### `app/scrapers/municipal/permit_scraper.py` — ⭐⭐⭐ Importante
**24.9 KB** — Scraper para permisos municipales de Chicago y Cook County. Extrae oportunidades de contratación locales que normalmente no aparecen en portales federales.

### `app/scrapers/saas/playwright_scraper.py` — ⭐⭐⭐ Importante
Scraper Playwright (headless browser) para sitios que requieren JavaScript para renderizar su contenido. Más lento que los scrapers HTTP directos, pero necesario para sitios como OpenGov.

---

## Base de Datos

### `app/database/qdrant_client.py` — ⭐⭐⭐⭐⭐ Crítico
Cliente Qdrant implementado como **Singleton**. Una sola instancia compartida entre todos los workers Celery (evita saturar el cluster con múltiples conexiones).

Métodos principales:
- `store_contracts_batch()` — Upsert de contratos en Qdrant (modo configurable).
- `check_existing_ids()` — Verifica qué IDs ya existen (para modo `incremental`).
- `clear_collection()` — Vacía la colección (modo `replace_all` — ⚠️ PELIGROSO).

> ⚠️ **No llamar a `store_contracts()` directamente** — usa `store_contracts_batch()` desde el pipeline. El método directo genera `stable_id`s con una función diferente, causando inconsistencias.

---

## Modelos y NLP

### `models/` — ⭐⭐⭐⭐⭐ Crítico
Artefactos ML serializados:
- `category_clf.pkl` (392 KB) + `category_vectorizer.pkl` (391 KB) — Clasificador de categoría de negocio. Precisión: 98.1%.
- `naics_clf.pkl` (118 MB) + `naics_vectorizer.pkl` (586 KB) — Clasificador de código NAICS. Precisión: 74%. Supera 100MB → requiere **Git LFS** para almacenar en GitHub.

> ⚠️ **Anclar versión scikit-learn en `requirements.txt`.** Los modelos `.pkl` son incompatibles entre versiones mayores de scikit-learn.

### `app/nlp/` — ⭐⭐⭐⭐ Muy importante
Motor NLP con spaCy para clasificación NAICS mejorada. Procesador en micro-batches configurable. Los artefactos spaCy se cargan desde el directorio `NLP_ARTIFACTS_DIR` (volumen montado en Render).

---

## Routers FastAPI

### `app/routers/` — ⭐⭐⭐⭐ Muy importante
Rutas organizadas por dominio:
- `hybrid_jobs.py` — `POST /hybrid-jobs` (crear job), `GET /csv-jobs` (listar), `GET /csv-jobs/{id}/results` (resultados).
- `schedules.py` — CRUD de programaciones de scraping. Consulta la BD para jobs con `next_run <= now`.
- `auth.py` — Login/logout con Firebase REST API. Gestiona cookie de sesión.

---

## Tareas Celery

### `app/tasks/job_tasks.py` — ⭐⭐⭐⭐⭐ Crítico
**30.5 KB** — Las tareas que ejecuta el worker de Celery. Función principal: `run_hybrid_pipeline_task()`.

Implementa:
- Checkpoints de progreso guardados en Redis (para recuperación tras reinicios).
- Actualización del estado del job (`created` → `running` → `completed`/`failed`).
- Envío de notificación email al terminar.
- Guardado del CSV output en Redis para acceso del servicio web.

---

## Utilidades

### `app/utils/stable_id.py` — ⭐⭐⭐⭐ Muy importante  
**Fuente de verdad para la generación de IDs únicos de contratos.** Usa URL canonizada + número de contrato para generar un hash SHA-256 determinístico. Todos los componentes deben usar esta función para garantizar consistencia en Qdrant.

### `app/utils/email_service.py` — ⭐⭐⭐ Importante
Servicio de email via Microsoft 365 SMTP. Envía notificaciones de completación/fallo de jobs con detalles del run.

### `app/utils/rate_limiter.py` — ⭐⭐⭐ Importante
Rate limiter para endpoints costosos. Previene que un usuario abuse de los endpoints de scraping o de la API de Qdrant.

### `app/utils/session_persistence.py` — ⭐⭐⭐ Importante
Manejo de persistencia de estado de jobs entre reinicios del worker. Lee/escribe checkpoints de progreso en Redis.

### `app/utils/agency_extractor.py` — ⭐⭐⭐ Importante
Módulo especializado para extraer el nombre de la agencia del texto de contratos usando NLP y heurísticas. Fundamental para el campo `agency` que afecta la clasificación de nivel de gobierno.

### `app/utils/pdf_ocr.py` — ⭐⭐⭐ Importante
Extracción de texto de PDFs de contratos (PyMuPDF + fallback OCR). Usado en la etapa de PDF Enrichment para SAM.gov.

---

## Tests

### `tests/conftest.py` — ⭐⭐⭐⭐ Muy importante
Fixtures compartidas de pytest: cliente HTTP de prueba, bypass de autenticación Firebase (mock), y mock del cliente Qdrant. Todo test debe usar estas fixtures.

### `tests/test_hybrid_jobs.py` — ⭐⭐⭐⭐ Muy importante
Pruebas del flujo principal end-to-end: crear job → verificar estado → obtener resultados. Es la prueba más representativa del sistema.

### `tests/test_nlp.py` — ⭐⭐⭐ Importante
Pruebas de precisión de los modelos ML. Corre predicciones sobre un conjunto de validación y falla si la precisión cae por debajo del umbral mínimo aceptable.

### `tests/test_auth.py` — ⭐⭐⭐ Importante
Pruebas del sistema de autenticación. Verifica que endpoints protegidos retornen 401 sin credenciales y 200 con credenciales válidas.

---

## Archivos de Configuración

### `pyproject.toml` — ⭐⭐⭐⭐ Muy importante
Configuración de Poetry (gestor de dependencias). Define versiones exactas de todas las dependencias y sus restricciones de compatibilidad. Usar `poetry install` para reproducir el entorno exacto de producción.

### `requirements.txt` — ⭐⭐⭐⭐ Muy importante
Versión exportada de dependencias para Render (que usa `pip`). Generado con `poetry export`. **No editar manualmente** — usar `poetry add` y re-exportar.

> ⚠️ **tiktoken** está anclado a una versión específica con wheels pre-compilados para `manylinux_2_17` (Render Linux). Cambiar esta versión puede romper el build.

---

## Scripts Legacy

### `app/scripts/legacy/` — ⭐⭐ Referencia histórica
Scripts one-shot de las primeras versiones del proyecto. Ya no son parte del flujo productivo. Mantenidos como referencia de cómo se realizaban las operaciones antes del pipeline unificado.

Incluye: `scrape_all_contracts.py`, `upload_to_qdrant.py`, `enrich_contracts.py`, `enrich_with_gpt.py`.
