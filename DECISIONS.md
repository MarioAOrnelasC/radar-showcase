# DECISIONS.md — RADAR
## Registro de Decisiones Técnicas y Justificaciones Arquitectónicas

Este documento registra las decisiones técnicas más relevantes tomadas durante el ciclo de desarrollo de RADAR, basado en el historial completo de commits y los documentos de handoff de sesiones de Devin AI. El objetivo es que cualquier nuevo integrante del equipo entienda el *por qué* detrás de cada cambio.

---

## ADR-001: Modos de Carga a Qdrant (Incremental / Upsert / Replace_All)

### Contexto
RADAR almacena contratos procesados en Qdrant. El sistema original siempre subía todos los contratos sin distinción, causando duplicados y tiempos de procesamiento innecesariamente largos en ejecuciones repetidas.

### Problema
- Volver a correr el pipeline completo sobre datos ya existentes re-subía todos los contratos, inflando la colección.
- No había forma de actualizar un contrato ya existente sin borrar y re-subir todos.
- La opción de "limpiar todo y volver a empezar" no tenía ningún control de seguridad.

### Decisión
Se implementaron 3 modos de carga configurables via la variable `qdrant_mode`:

| Modo | Comportamiento | Caso de Uso |
|---|---|---|
| `incremental` | Omite IDs ya existentes en la colección | Actualizaciones diarias sin duplicados |
| `upsert` | Actualiza existentes y agrega nuevos | Re-procesar contratos con datos corregidos |
| `replace_all` | Vacía la colección y sube todo | Reconstrucción total (PELIGROSO) |

### Problema de Seguridad Identificado — replace_all
La implementación original pasaba `confirm=True` de forma hardcodeada, sin requerir ninguna confirmación adicional del usuario. Esto hacía posible borrar toda la colección de producción con un solo click.

**Mitigación implementada:** Se documentó este riesgo y se requirió revisar el panel de frontend para que el modo `replace_all` solo sea visible como opción cuando esté explícitamente habilitado.

**Commits Relacionados:** `306f5e3`, `738cc34`, `b648100`

---

## ADR-002: Refactorización del Monolito — De `main.py` a Routers Modulares

### Contexto
FastAPI permite definir todas las rutas directamente, por lo que toda la lógica de RADAR (rutas HTTP, negocio, queries a BD, scraping) creció hasta superar las **2,500 líneas** en un único `main.py`.

### Problema
- Tests unitarios no podían aislar funcionalidades sin cargar el módulo completo.
- Conflictos de merge frecuentes entre colaboradores en el mismo archivo.
- Añadir un endpoint requería navegar cientos de líneas para encontrar el lugar correcto.

### Alternativas Consideradas

| Opción | Decisión |
|---|---|
| Mantener monolito, documentar mejor | Descartada: deuda técnica insostenible |
| Clean Architecture estricta (controllers/services/repos) | Descartada: exceso de abstracción para el tamaño del equipo |
| **Separar por dominio (APIRouter)** ✅ | Elegido: equilibrio entre organización y pragmatismo |

### Implementación
Se utilizó el patrón `APIRouter` nativo de FastAPI. Cada dominio obtuvo su propio archivo en `app/routers/`:
- `routers/auth.py` — Autenticación JWT
- `routers/scrapers.py` — Endpoints de scraping SAM.gov, Grants.gov
- `routers/hybrid_jobs.py` — Jobs combinados de scraping + ML
- `routers/schedules.py` — Gestión de tareas Celery programadas

**Problema Encontrado — Imports circulares:**
Al separar los módulos, el módulo A importaba de B y B importaba de A.
**Solución:** Se creó `app/core/dependencies.py` como módulo central de dependencias compartidas (cliente de BD, configuración) que todos los routers importan sin generar ciclos.

**Commits Relacionados:** `40fb7f8`, `26e822f`, `4faef00`

---

## ADR-003: Integración de Modelos ML (Reemplazo Híbrido de GPT-4)

### Contexto
RADAR clasificaba contratos enviando su texto completo a OpenAI GPT-4 para determinar la vertical de negocio y el código NAICS. El costo y la latencia eran prohibitivos a escala.

### Problema Cuantificado
- **Costo:** Cada clasificación consumía ~1,500 tokens de GPT-4. Al procesar miles de contratos semanalmente, el costo era significativo y creciente.
- **Latencia:** 2-5 segundos por contrato hacía los batch jobs de scraping extremadamente lentos.
- **Dependencia externa:** Cualquier fallo en la API de OpenAI paralizaba completamente la clasificación.

### Evaluación de Alternativas

| Opción | Precisión | Costo | Latencia | Decisión |
|---|---|---|---|---|
| GPT-4 (actual) | ~78% (dominio específico) | Alto | 2-5s | Reemplazar |
| Fine-tuning GPT-3.5 | ~75–80% | Medio | 1-2s | Descartado |
| Random Forest | ~82% (estimado) | Bajo | ms | Descartado (50MB de modelo) |
| BERT/Transformers | ~90% (estimado) | Bajo (GPU requerida) | ms | Descartado (RAM excesiva en Render) |
| **scikit-learn TF-IDF + SGDClassifier** ✅ | **98.9% categoría / 86.3% NAICS (6d)** | Despreciable | ms | Adoptado |

### Por qué scikit-learn superó a GPT-4 en este dominio
Los contratos gubernamentales tienen vocabulario muy específico y estandarizado (términos del FAR, nomenclatura de SAM.gov, acrónimos de agencias). Un **SGDClassifier** (SVM lineal con `loss='hinge'`, entrenado sobre vectores TF-IDF de hasta 20,000 features) entrenado con datos históricos de contratos resultó más preciso, más rápido y más económico que GPT-4 para esta tarea de clasificación de dominio cerrado. El TF-IDF aprende exactamente los co-ocurrencias de terminología federal; GPT-4 tiende a generalizar en lugar de especializarse.

### Implementación y Problemas Encontrados

**Problema 1 — Carga costosa de modelos `.pkl`:**
Los archivos de modelo pesan varios cientos de MB. Cargarlos en cada request mantenía latencia alta.
**Solución:** Patrón de **Singleton por lifespan** con `@app.on_event("startup")`. Los modelos se cargan una sola vez al arrancar y se almacenan en `app.state.ml_models`.

**Problema 2 — Desajuste de versiones de scikit-learn:**
Los modelos `.pkl` se serializaron con una versión específica. Si el entorno de producción tenía una versión diferente, se lanzaban `UserWarnings` que podían convertirse en errores.
**Solución:** Se ancló la versión exacta en `requirements.txt` (`scikit-learn==X.Y.Z`) para garantizar paridad entre entornos de entrenamiento y producción.

**Commits Relacionados:** `2612853`, `e2c3762`, `1bdeb01`, `62a14e4`, `938d65d`

---

## ADR-004: Mejoras a los Scrapers (SAM.gov y Grants.gov)

### Contexto
Los scrapers originales de SAM.gov extraían datos básicos y no tenían paginación, limitando la cobertura a los primeros resultados de la API. Tampoco existía un scraper para Grants.gov (financiamiento federal de grants/subvenciones).

### Implementación — SAM.gov
- Se implementó paginación de hasta 1,000 resultados por consulta.
- Se mejoró la extracción de campos: descripción completa, códigos NAICS múltiples, estado del contrato (activo/cerrado/vencido).
- Se añadió clasificación automática de nivel de gobierno (federal/state/local) y tipo de oportunidad (solicitation/grant/award).

### Implementación — Grants.gov (nuevo scraper)
- Se creó scraper desde cero para obtener grants federales.
- Se extrajeron campos específicos de grants: códigos CFDA, categorías de programa, requisitos de elegibilidad.
- Se unificó el modelo de datos para que contratos y grants usen la misma estructura en Qdrant.

### Problema Encontrado — Comparación de fechas con timezone
El scraper de SAM.gov comparaba `datetime.now()` (sin timezone) con fechas del API de SAM.gov (con timezone UTC), resultando en una excepción `TypeError: can't compare offset-naive and offset-aware datetimes`.
**Solución:** Se usó `datetime.now(timezone.utc)` en lugar de `datetime.now()` para garantizar comparabilidad.

**Commits Relacionados:** `1f165c5`, `a032e21`, `c378f43`, `8e578dc`, `639adb9`

---

## ADR-005: Pipeline Robusto — Manejo de Errores por Fila en CSV

### Contexto
El pipeline de escritura de CSVs fallaba completamente si cualquier registro individual tenía un campo inesperado o valor nulo. Esto resultaba en outputs vacíos incluso cuando 99% de los registros eran válidos.

### Problema
Un solo contrato con un campo mal formado abortaba la escritura de todo el batch, perdiendo cientos de registros válidos.

### Decisión
Se implementó manejo de errores **por fila** en `store.py`: cada registro se escribe de forma independiente con su propio `try/except`. Si un registro falla, se loggea el error y el pipeline continúa con el siguiente registro.

También se separaron los outputs de contratos y grants en archivos CSV distintos con columnas apropiadas para cada tipo (contratos incluyen NAICS, grants incluyen CFDA).

**Commits Relacionados:** `02b9c2a`, `685f96b`, `53b665e`

---

## ADR-006: Suite de Tests Automatizados con Pytest

### Contexto
El proyecto RADAR no tenía ninguna prueba automatizada. Cualquier refactorización requería pruebas manuales extensas.

### Estructura Implementada

```
tests/
├── conftest.py           # Fixtures compartidas (app test client, auth bypass, mock BD)
├── test_auth.py          # Pruebas de registro, login y tokens JWT
├── test_scrapers.py      # Pruebas de scrapers (con mocks HTTP)
├── test_hybrid_jobs.py   # Pruebas de flujos de scraping + clasificación ML
├── test_schedules.py     # Pruebas de API de tareas programadas
└── test_nlp.py           # Pruebas de precisión de modelos ML
```

**Problema 1 — Incompatibilidad FastAPI/Starlette:**
`pip` resolvía versiones incompatibles de FastAPI y Starlette. El `TestClient` de Starlette lanzaba errores de importación.
**Solución:** Se fijaron versiones compatibles explícitas en `requirements-dev.txt`, consultando la matriz de compatibilidad en el CHANGELOG oficial de FastAPI.

**Problema 2 — `pywin32` en entornos Linux:**
Celery en Windows importa `pywin32` para gestión de procesos. Al correr tests en CI (Linux), la importación fallaba.
**Solución:** `try/except ImportError` en el módulo de inicialización de Celery para ignorar `pywin32` en sistemas no-Windows.

**Commits Relacionados:** `15ce141`, `685494a`, `95c6385`

---

## ADR-007: Patrón Singleton para QdrantDatabase

### Contexto
Con workers de Celery, cada worker creaba su propia instancia del cliente Qdrant al recibir una tarea. Bajo carga, esto generaba decenas de conexiones simultáneas al cluster, superando los límites del plan gratuito.

### Solución
Se implementó el patrón Singleton para la clase `QdrantDatabase`:
- La instancia del cliente Qdrant se crea una única vez y se reutiliza en todas las llamadas.
- Se documentó que `QdrantDatabase.store_contracts()` no debe llamarse directamente (usa IDs generados por una función diferente a la del pipeline principal, generando inconsistencias).

**Commit Relacionado:** `5e185a8`

---

## ADR-008: Deprecación de Pipelines Legacy y Reorganización de Scripts

### Contexto
El directorio raíz había acumulado 15+ scripts independientes creados para tareas one-shot de carga de datos. Su presencia en el root confundía sobre cuáles scripts eran parte del flujo actual.

### Decisión
1. **Scripts standalone** → Movidos a `/legacy/` con un `README_LEGACY.md` explicando su propósito y por qué fueron reemplazados.
2. **Pipelines ETL/CSV** → Eliminados. La lógica útil fue absorbida por los nuevos routers.
3. **Módulo `observability.py`** → Añadido para reemplazar los logs ad-hoc de los scripts legacy, registrando métricas de pipeline (contratos procesados por hora, tasa de error de clasificación, latencia media de scraping).

**Commits Relacionados:** `26e822f`, `4faef00`, `3b8eaaa`

---

## ADR-009: Actualización de Límites de Tiempo en Celery

### Problema
Los timeouts de Celery (`task_soft_time_limit`, `task_time_limit`) estaban configurados con valores conservadores de iteraciones anteriores cuando los jobs eran más pequeños. Al procesar lotes grandes de SAM.gov (hasta 1,000 contratos con paginación), los workers eran terminados prematuramente con `SoftTimeLimitExceeded`.

### Decisión
Actualización en `config.py`:
- `task_soft_time_limit`: 300s → 900s (15 minutos).
- `task_time_limit`: 360s → 1,200s (20 minutos).

## ADR-010: Arquitectura Celery + Redis — Resolución de Crashes por OOM en Producción

### Contexto
El pipeline de scraping de RADAR (fetch de páginas web, extracción LLM, generación de embeddings, upload a Qdrant) es una operación masiva que puede tardar decenas de minutos para un batch grande. La implementación original ejecutaba todo esto de forma síncrona dentro del proceso de FastAPI.

### Problema — OOM y Timeouts en Producción (Render / Fly.io)
Al correr el pipeline completo en el mismo proceso que el servidor HTTP:
1. El pipeline consumía toda la RAM disponible del servidor mientras procesaba cientos de contratos simultáneamente (fetch paralelo + LLM + embeddings).
2. El sistema operativo terminaba el proceso por **OOM (Out of Memory)**, derribando también el servidor HTTP.
3. Incluso sin llegar al OOM, el pipeline tardaba más que el timeout de FastAPI, causando errores 504.
4. Si el proceso se reiniciaba, **todos los jobs en progreso se perdían** sin ningún mecanismo de recuperación.

**Evidencia en commits:**
- `3db0fe9` — *"Switch back to original Render backend URL due to Fly.dev memory issues"* — El servidor en Fly.io colapsó por OOM y se tuvo que hacer rollback.

### Alternativas Consideradas

| Opción | Problema |
|---|---|
| Aumentar RAM del servidor | Caro, no escala, no resuelve el problema de fondo |
| `asyncio` + `aiohttp` en el mismo proceso | El proceso sigue compartiendo memoria con el servidor HTTP; OOM sigue siendo posible |
| FastAPI `BackgroundTasks` | Corre en el mismo proceso del servidor; OOM y timeouts persisten |
| **Celery + Redis** ✅ | Workers en procesos separados con memoria propia; broker Redis persiste jobs entre reinicios |

### Decisión — Arquitectura de Workers Independientes

Se implementó una arquitectura de **dos procesos en Render** completamente independientes:

```
┌─────────────────────────────────────────────────────────┐
│ Render Service 1 – Web (FastAPI)                        │
│                                                         │
│  POST /hybrid-jobs → Celery.send_task() → HTTP 202     │
│  GET  /csv-jobs/{id}/status → Redis lookup             │
└────────────────────────┬────────────────────────────────┘
                         │ Redis Broker
                         │ (CloudAMQP / Upstash Redis)
                         │
┌────────────────────────▼────────────────────────────────┐
│ Render Service 2 – Celery Worker                        │
│                                                         │
│  [Task] Ejecuta pipeline completo:                      │
│    Scraping → LLM Extraction → Embeddings → Qdrant     │
│  Guarda progreso en Redis (checkpoints)                 │
│  Guarda output CSV en Redis (compartido con web)        │
└─────────────────────────────────────────────────────────┘
```

### Implementación y Problemas Encontrados

**Problema 1 — Jobs en estado 'created' que nunca se ejecutaban:**
Después de la integración inicial, los jobs se creaban en la BD pero el worker de Celery nunca los tomaba.
**Causa:** Se había configurado routing explícito de tareas hacia una queue específica inexistente en el broker.
**Solución:** Se eliminó el routing innecesario, permitiendo que Celery use la queue `default`.
**Commit:** `8d4ab3e` — *"Remove task queue routing that caused jobs to stay in 'created' state"*

**Problema 2 — Output CSV inaccesible desde el servicio web:**
El worker de Celery generaba los CSVs en su propio sistema de archivos efímero (Render). Cuando el usuario intentaba descargar el resultado desde el servicio web, el archivo no existía.
**Solución:** Se almacenan los archivos de output en **Redis** (serialización base64) con TTL de 24 horas, accesibles por ambos servicios.
**Commits:** `137154d`, `ca56915`, `c4eaf76`

**Problema 3 — Jobs perdidos al reiniciar el worker:**
Al reiniciar el proceso worker (redeploy, OOM), los jobs en progreso se perdían.
**Solución:** Se implementó infraestructura de **checkpoints**: el worker guarda el progreso parcial en Redis periódicamente. Si se reinicia, puede retomar desde el último checkpoint.
**Commit:** `2e7e274` — *"Prevent job rollback on worker restart with checkpoint infrastructure"*

**Problema 4 — Re-entrega de tareas largas por timeout del broker:**
Celery tiene un `visibility_timeout`: si un worker no confirma la tarea en ese tiempo, el broker la re-entrega a otro worker (causando ejecución doble). Los jobs de scraping masivo tardaban más que el timeout por defecto.
**Solución:** Se incrementó el `visibility_timeout` de Redis en `config.py` para reflejar la duración máxima esperada de los jobs.
**Commit:** `7197a7d` — *"Increase Redis visibility timeout to prevent task re-delivery for long jobs"*

**Problema 5 — Logs insuficientes para debuggear workers remotos:**
Al correr en Render, los logs del worker eran escasos. Debuggear por qué un job se congelaba era prácticamente imposible.
**Solución:** Se añadió logging detallado en puntos críticos del pipeline (inicio de cada etapa, número de contratos procesados, errores por registro).
**Commits:** `491c181`, `3f0b7d7`, `d6ca92d`

### Paralelismo Configurable via Variables de Entorno

El número de workers paralelos para extracción LLM y fetch de páginas se hizo configurable via variables de entorno:
- `PARALLEL_LLM_WORKERS` (default: 5) — Workers paralelos para extracción GPT.
- `PARALLEL_FETCH_WORKERS` (default: 10) — Workers paralelos para scraping HTTP.

Esto permite ajustar el paralelismo según los recursos de la instancia en Render sin necesidad de redeploy.
**Commits:** `23d95cd`, `9b42a1a`

### Resultado
- **Zero crashes OOM** en producción después de la migración.
- Jobs de scraping de hasta 1,000 contratos completándose en background sin interferir con el servidor HTTP.
- Persistencia de jobs y outputs entre reinicios gracias a Redis.

---

## ADR-011: Sistema de Autenticación — Firebase REST API + Sesiones

### Contexto
RADAR es una herramienta interna para operadores de IHCC. Necesita autenticación para proteger los endpoints de scraping y gestión de jobs (que consumen recursos costosos de GPT-4 y Qdrant).

### Decisión de Stack
Se adoptó **Firebase Authentication via REST API** en lugar de un sistema JWT propio, por las siguientes razones:
- Sin necesidad de gestionar almacenamiento de contraseñas, salting, o expiración de tokens manualmente.
- El token ID de Firebase puede verificarse en el backend con el Admin SDK sin llamadas de red adicionales.
- Consistencia con CORAMA, reduciendo la curva de aprendizaje entre proyectos.

### Implementación

**login** (`routers/auth.py`): Recibe email/password, los envía a la REST API de Firebase, obtiene un `idToken` de firma JWT. El token se guarda en la cookie de sesión del servidor (HTTPOnly, SameSite=Lax).

**Problema — Cookie de sesión no enviada en paths protegidos:**
Las cookies de sesión de FastAPI no se enviaban en requests a rutas distintas de `/`. Esto causaba que usuarios autenticados recibieran 401 en endpoints como `/hybrid-jobs`.
**Solución:** Se configuró `path='/'` explícitamente en la cookie de sesión para que sea enviada en todas las rutas.
**Commit:** `08e6fb5` — *"Add path='/' to session cookie to ensure it's sent with all requests"*

**Commits clave:** `54fde6c`, `c9d68bd`, `93a0def`, `db43419`

---

## ADR-012: Fix de Dependencia tiktoken para Builds en Render (Linux)

### Contexto
El proyecto usa `tiktoken` (librería de OpenAI) para contar tokens antes de enviar texto a la API GPT. `tiktoken` requiere compilar código Rust en el momento de instalación.

### Problema
En el entorno de build de Render (Linux `manylinux`), la versión de `tiktoken` disponible en PyPI no tenía wheels pre-compilados para la arquitectura de Render. El build de `pip install` intentaba compilar desde código fuente, fallando por dependencias de compilación no disponibles en el entorno de Render.

**Error observado:** `error: could not find a version of tiktoken that satisfies the requirement`

### Solución
Tras investigar las releases de tiktoken en PyPI, se identificó que:
- `tiktoken==0.5.2` tiene wheels `manylinux_2_17` compatibles con Render.
- `tiktoken==0.8.0` tiene wheels pre-compilados para la versión de Python usada.

Se actualizó `requirements.txt` a la versión con wheel disponible y se verificó el build en Render.

**Commits:** `7227dbf`, `79dc980`

---

## ADR-013: Motor NLP con spaCy — Reentrenamiento con 34K Ejemplos

### Contexto
Para mejorar la predicción de códigos NAICS más allá de los clasificadores TF-IDF, se integró un motor NLP basado en **spaCy** que combina tokenización lingüística con el clasificador de texto (`textcat`).

### Proceso de Reentrenamiento
1. Se recopiló un dataset de **34,000 contratos etiquetados** con sus códigos NAICS reales.
2. Se entrenaron artefactos NLP compactos (<65MB) compatibles con el plan de memoria de Render.
3. Se hizo configurable la ruta de los artefactos via la variable `NLP_ARTIFACTS_DIR`, lo que permite cargar modelos desde un volumen montado en Render sin hard-codear paths.

### Implementación Técnica
El motor NLP fue diseñado como un **micro-batch processor**: en lugar de procesar un contrato a la vez (latencia alta) o todos de una vez (memoria alta), procesa lotes de N contratos configurables, balanceando memoria y velocidad.

Se añadió también un script de evaluación masiva (`eda_prediction_script.py`) para correr predicciones sobre CSVs completos y medir la precisión antes del deploy.

**Commits:** `2557dcf`, `46b7694`

---

## ADR-014: Enriquecimiento PDF de Contratos SAM.gov

### Contexto
Muchos contratos en SAM.gov tienen información crítica (presupuesto exacto, fechas clave, requisitos técnicos) publicada en documentos PDF adjuntos, no en los campos estructurados de la API. Los scrapers iniciales solo extraían los campos de la API.

### Implementación
Se añadió una etapa de **PDF Enrichment** al pipeline:
1. El scraper de SAM.gov detecta contratos con documentos PDF adjuntos (`8cc15d9` — *"Change SAM_GOV_FETCH_PDFS default to true"*).
2. Los PDFs se descargan y se pasan al módulo `pdf_enrichment/`.
3. Se usa GPT-4 para extraer campos faltantes del texto del PDF (presupuesto, fechas de período de performance, nombre del oficial contratante, etc.).
4. Los campos extraídos enriquecen el registro del contrato antes de almacenarlo en Qdrant.

**Commits:** `0886390`, `64732a3`, `6609cbe`

---

## ADR-015: Notificaciones por Email para Completación/Fallo de Jobs

### Contexto
Los jobs de scraping masivo pueden tardar entre 15 y 60 minutos. Los usuarios necesitaban refrescar manualmente la página para saber si su job había terminado.

### Implementación
Se añadió un sistema de notificaciones por email al completar o fallar un job:
- **Email de éxito:** Incluye el número de contratos procesados, tiempo total de ejecución, errores por registro, y un link directo a los resultados.
- **Email de error:** Incluye el traceback completo del error para facilitar el debugging.
- Se migró el servicio de email de Gmail SMTP a **Microsoft 365 SMTP** para mayor fiabilidad y límites más altos de envío.
- Se añadió protección contra duplicados: si un job emite el estado "completed" más de una vez (por reintentos de Celery), solo se envía un email.

**Commits:** `cef7a64`, `5f31d95`, `0ed2efe`, `ad3297e`

---

## ADR-016: Sistema de Programación de Scrapers con Base de Datos

### Contexto
Los scrapers iniciales se activaban manualmente por los operadores de IHCC. Esto requería que alguien iniciara sesión y activara el scraping cada vez, lo cual no era sostenible para monitoreo continuo de contratos.

### Implementación
Se diseñó e implementó un sistema completo de programación con persistencia en base de datos:

**Fase 1 — Modelo de datos:**
- Tabla `scrape_sources`: Define las fuentes (SAM.gov, Grants.gov, Illinois, Chicago, Cook County) con sus configuraciones.
- Tabla `scheduled_jobs`: Almacena las programaciones con `cron_expression`, `next_run`, y `last_run`.
- Panel de administración UI para gestionar fuentes y programaciones.

**Fase 2 — Scheduler:**
- Un proceso de scheduling independiente verifica en la BD qué jobs deben ejecutarse (basado en `next_run`).
- Al completar un job, actualiza `last_run` y calcula el próximo `next_run` basado en la expresión cron.

**Fase 3 — Jitter y Monitoreo:**
- Se añadió **jitter aleatorio** (±n segundos) al tiempo de ejecución para evitar que múltiples fuentes hagan sus requests simultáneamente al mismo tiempo, distribuyendo la carga.
- Se añadió UI de progreso en tiempo real que muestra el estado de los scrapers activos.

**Commits:** `05bb1d8`, `76a087f`, `c1fd0e1`, `60e8aff`

---

## ADR-017: Almacenamiento de Modelos ML — Git LFS sobre Alternativas

### Contexto
Los artefactos ML del proyecto (clasificadores scikit-learn y vectorizadores TF-IDF) son archivos binarios de gran tamaño:
- `naics_clf.pkl` — **118 MB** (supera el límite de 100 MB de GitHub)
- `naics_vectorizer.pkl` — 586 KB
- `category_clf.pkl` — 392 KB
- `category_vectorizer.pkl` — 391 KB

GitHub rechaza archivos mayores a 100 MB con un error `pre-receive hook declined`, lo que bloqueó el push inicial del repositorio.

### Alternativas Consideradas

| Opción | Costo | Desventajas | Decisión |
|---|---|---|---|
| **Omitir modelos del repo** | Gratuito | Requiere proceso manual de descarga; no reproducible sin documentación adicional | Descartada |
| **DVC + S3/GCS** | **Pago** — S3 ~$0.023/GB/mes + egress; mínimo ~$5–15/mes según uso | Infraestructura adicional, credenciales separadas, curva de aprendizaje alta | Descartada |
| **DVC + Azure Blob** | **Pago** — Similar a S3; plan gratuito de Azure limitado a 12 meses | Igual que S3, con lock-in adicional al ecosistema Azure | Descartada |
| **Hugging Face Model Hub** | Gratuito para repos públicos; **privados desde $9/mes** | Diseñado para modelos DL; no es fit natural para scikit-learn; expone modelos propietarios si se usa plan gratuito | Descartada |
| **MLflow / Weights & Biases** | **W&B gratuito limitado** (100 GB); MLflow requiere servidor propio o Databricks (~$$$) | Sobreingeniería; costo escalaría con cada reentrenamiento | Descartada |
| **Git LFS** ✅ | **Gratuito** en GitHub (1 GB storage incluido) | Cuota de 1 GB/mes de bandwidth en plan gratuito | **Adoptado** |

### Por qué Git LFS fue la mejor opción

La decisión centró en **fricción mínima y costo cero** para un proyecto aún en etapa pre-revenue:

1. **Costo sostenible:** Todas las alternativas viables (DVC + S3/GCS, W&B, Hugging Face privado) implican costos mensuales recurrentes de entre $5 y $50+ USD/mes. Para un proyecto que aún no genera ingresos, asumir ese costo fijo solo para almacenar modelos es injustificable. Git LFS es **completamente gratuito** dentro de los límites del plan de GitHub.

2. **Transparencia total:** Después de `git lfs install` y definir `*.pkl` como archivos LFS en `.gitattributes`, el resto del flujo (commit, push, pull, clone) no cambia para ningún colaborador. Los modelos se descargan automáticamente al hacer `git pull`.

3. **Sin infraestructura adicional:** DVC y las soluciones de cloud storage manual requieren configurar buckets S3/Azure, gestionar credenciales de acceso por cada colaborador, y documentar el proceso de descarga fuera del repo. Git LFS usa las mismas credenciales de GitHub ya existentes.

4. **Versionado automático:** Cada `git commit` que modifica un modelo genera automáticamente una nueva versión en LFS. Se puede hacer `git checkout <hash>` para recuperar exactamente los modelos de cualquier punto en el historial, sin proceso adicional.

5. **Escala adecuada al problema:** DVC, MLflow y similares están diseñados para equipos con pipelines de reentrenamiento continuo y docenas de experimentos concurrentes. Para modelos re-entrenados ocasionalmente (cuando se acumulan suficientes datos nuevos), Git LFS es proporcionado y evita over-engineering costoso.

### Implementación

```bash
# Instalar Git LFS
git lfs install

# Trackear archivos .pkl
git lfs track "*.pkl"

# Esto crea/modifica .gitattributes:
# *.pkl filter=lfs diff=lfs merge=lfs -text

git add .gitattributes
git add models/
git commit -m "feat: add ML models via Git LFS"
git push origin main
```

El archivo `.gitattributes` en el repositorio define qué extensiones se gestionan como LFS. Cualquier colaborador que clone el repo obtiene automáticamente los punteros LFS; los archivos reales se descargan al hacer `git lfs pull`.

### Limitaciones Conocidas
- El plan gratuito de GitHub LFS incluye **1 GB de almacenamiento y 1 GB/mes de ancho de banda**. Si los modelos crecen significativamente (ej. modelos de lenguaje completos), habrá que migrar a un LFS externo o a DVC + S3.
- Al clonar en CI/CD sin soporte LFS, los archivos `.pkl` son punteros de texto, no los modelos reales. Esto es aceptable siempre que el CI no requiera correr las predicciones ML.
