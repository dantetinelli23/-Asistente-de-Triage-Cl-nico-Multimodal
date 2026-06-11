# 🏥 Asistente de Triage Clínico Multimodal

> Pipeline de automatización con IA construido en **n8n self-hosted**: recibe consultas de pacientes por texto, audio o imagen, las responde con un triage clínico fundamentado en protocolos reales (RAG), incorpora revisión médica humana en los casos críticos, se evalúa a sí mismo y vigila su propia calidad en producción.

---

## 📋 Tabla de contenidos

- [Qué problema resuelve](#-qué-problema-resuelve)
- [Stack técnico](#-stack-técnico)
- [Arquitectura general](#-arquitectura-general)
- [Procesamiento multimodal](#-procesamiento-multimodal-audio--imagen--texto)
- [Sistema RAG](#-sistema-rag)
- [Evaluación automática (LLM-as-judge)](#-evaluación-automática-llm-as-judge)
- [Human-in-the-Loop](#-human-in-the-loop-hitl)
- [Fallbacks y políticas de espera](#-fallbacks-y-políticas-de-espera)
- [Observabilidad y trazabilidad](#-observabilidad-y-trazabilidad)
- [Manejo de errores](#-manejo-de-errores)
- [Workflow auto-mejorable](#-workflow-auto-mejorable)
- [Seguridad y privacidad](#-seguridad-y-privacidad)
- [Decisiones de diseño destacadas](#-decisiones-de-diseño-destacadas)
- [Limitaciones conocidas](#-limitaciones-conocidas)

---

## 🎯 Qué problema resuelve

El paciente no sabe juzgar la gravedad de sus propios síntomas, y esa incertidumbre casi siempre se resuelve mal: el que tiene algo leve satura la guardia "por las dudas", y el que tiene algo grave lo minimiza y pierde tiempo. El sistema actúa como el **enfermero de triage de una guardia**: no diagnostica ni trata, *ordena y prioriza* antes de que la persona llegue.

El paciente describe sus síntomas (escribe, manda una nota de voz o saca una foto) y recibe: **nivel de urgencia + área afectada + recomendación inicial**, fundamentada en protocolos clínicos reales. Los casos de urgencia alta no salen automáticamente: los revisa un médico.

> ⚠️ **El sistema NO diagnostica.** Entrega una orientación preliminar basada en evidencia, y lo aclara explícitamente en cada respuesta. Esta restricción condiciona todas las decisiones de diseño.

---

## 🛠 Stack técnico

| Capa | Herramienta | Rol |
|------|-------------|-----|
| Orquestación | **n8n** (self-hosted) | Motor de workflows |
| Canal | **Telegram** | Entrada multimodal del paciente (webhook) |
| LLM principal | **Groq** (`llama-3.3-70b-versatile`) | Agente de triage |
| LLM fallback | **OpenAI / OpenRouter** | Respaldo ante fallo del principal |
| LLM evaluador | **Anthropic** (`claude-sonnet-4.6`) | Jurado de calidad |
| STT | **Groq Whisper** (`whisper-large-v3-turbo`) | Transcripción de audio |
| Embeddings | **Cohere** (`embed-multilingual-v3.0`) | Vectorización (1024-dim) |
| Reranking | **Cohere Rerank** | Segundo filtro de relevancia |
| Vector DB | **Pinecone** | Búsqueda semántica |
| Document parsing | **LlamaCloud / LlamaParse** | Ingesta de protocolos (PDF/DOCX) |
| Métricas de negocio | **Airtable** | Tablas `CONSULTAS` y `LOGS_SISTEMA` |
| Tracing técnico | **LangSmith** | Latencia, tokens, costo por run |
| Análisis | **NotebookLM** | Conclusiones sobre datos exportados |

---

## 🏗 Arquitectura general

El sistema está dividido en **tres workflows independientes** que comparten la base de conocimiento. La separación es intencional: resuelven problemas distintos y se ejecutan en momentos distintos.

```
┌──────────────────────────────────────────────────────────────────┐
│  FLUJO A — INGESTA  (one-shot por documento)                       │
│  Form → LlamaParse → polling → chunking(1000/200) → Cohere → Pinecone │
└──────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────┐
│  FLUJO B — CONSULTA  (real-time, por paciente)                     │
│                                                                    │
│  Telegram Trigger (webhook)                                        │
│        │                                                           │
│   ┌────▼─────┐  Switch por tipo de mensaje                         │
│   │  audio   │──► STT(Whisper) ─► quality gate ─┐                  │
│   │  imagen  │──► agente multimodal ────────────┤                  │
│   │  texto   │──► directo ─────────────────────┤                   │
│   └──────────┘                                  │                  │
│                            (agente de triage + RAG)                │
│                                    │                               │
│                              ┌─────▼──────┐                        │
│                              │ Normalizar │ ← unifica las 3 ramas  │
│                              └─────┬──────┘                        │
│                                    ▼                               │
│                          Evaluador (LLM juez)                      │
│                                    ▼                               │
│                          Comparación triage vs juez                │
│                                    ▼                               │
│                             ¿Escalamos?                            │
│                           ┌────────┴────────┐                      │
│                          SÍ                NO                      │
│                           ▼                 ▼                      │
│                    HITL (médico)     respuesta autónoma            │
│                           │                 │                      │
│                           └────────┬────────┘                     │
│                                    ▼                               │
│                       respuesta al paciente (Telegram)            │
│                                                                    │
│   ▸ cada checkpoint dispara un Logger → LOGS_SISTEMA               │
│   ▸ cada llamada a LLM queda registrada en LangSmith              │
└──────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────┐
│  FLUJO C — AUTO-MEJORA  (scheduled, diario)                        │
│  Schedule → lee CONSULTAS → calcula KPIs → ¿degradación? → alerta  │
└──────────────────────────────────────────────────────────────────┘
```

**Clave del diseño:** las tres modalidades convergen en un único nodo `Normalizar` que las deja con un esquema común (`rama`, `relato_paciente`, `triage_text`, `calidad_audio`, etc.). A partir de ahí, **el resto del pipeline es agnóstico al canal de origen**. Esto hizo que sumar la tercera modalidad (texto) fuera trivial: toda la cadena posterior ya estaba lista para absorberla.

---

## 🎙 Procesamiento multimodal (audio · imagen · texto)

El cerebro del sistema razona sobre **texto**. El trabajo de cada rama es llevar su formato a algo que el agente entienda, y cada una lo hace distinto según lo que ese dato necesita:

```
TEXTO   → ya es texto                          → directo al agente
AUDIO   → STT (Whisper) traduce voz a texto    → al agente
IMAGEN  → modelo multimodal "ve" la foto        → al agente (sin transcribir)
```

### Routing
Un nodo **Switch** inspecciona el mensaje de Telegram:
- `message.voice` presente → rama **audio**
- `message.photo` presente → rama **imagen**
- `message.text` presente → rama **texto**

Un solo bot, ramas internas. Esto mantiene consistente el manejo de errores y la trazabilidad entre las tres.

### Rama de audio + quality gate
La nota de voz se descarga, se renombra al formato correcto y se envía por **REST a Groq Whisper** (`temperature: 0`, `language: es`, `response_format: verbose_json`).

Lo interesante es que **no se confía ciegamente en la transcripción**. Un nodo calcula métricas objetivas a partir de los `segments` que devuelve Whisper:

```js
// señales de calidad derivadas de Whisper
avg_logprob      // confianza promedio de la transcripción
logprob_peor     // confianza del peor segmento
no_speech_prob   // probabilidad de que sea ruido y no habla
compression_ratio // detección de texto repetitivo/corrupto
```

Con esas señales clasifica el audio en `alta / media / baja`. Si la calidad es baja, **el sistema pide regrabar** en vez de hacer triage sobre una transcripción dudosa. Estas señales viajan luego hasta el evaluador y las métricas.

### Rama de imagen
El modelo multimodal recibe el binario de la imagen (descargado automáticamente por el trigger) + el caption del paciente, y produce el triage describiendo lo que ve. No hay paso de transcripción: "ver" ya viene incluido en el modelo.

---

## 🔍 Sistema RAG

**Por qué RAG:** un LLM solo (a) no conoce *estos* protocolos y (b) cuando no sabe, **alucina**. En salud, una respuesta inventada que suena segura es inaceptable. RAG fuerza al modelo a responder *a libro abierto*, apoyándose en la base de conocimiento propia. Por eso cada respuesta puede citar de qué protocolo salió.

### Pipeline de ingesta
```
PDF/DOCX
  → LlamaParse (async): documento complejo → Markdown estructurado
  → polling (Wait + IF hasta COMPLETED): el parsing es asincrónico
  → Recursive Character Text Splitter: chunks de 1000, overlap 200
  → Cohere embeddings (multilingual-v3.0): cada chunk → vector 1024-dim
  → Pinecone (insert): vector + texto original como metadata
```

**Decisiones técnicas:**

- **`chunkSize: 1000` / `chunkOverlap: 200`** — el overlap evita que una afirmación crítica (ej. *"si la fiebre supera 39° derivar a guardia"*) quede partida entre dos chunks y se pierda en la búsqueda. Es la "posta de relevos": la zona de empalme aparece completa en al menos un chunk.
- **Embeddings multilingües** — los protocolos están en español formal y los pacientes escriben en coloquial argentino. El modelo entiende que *"me duele la panza"* y *"dolor abdominal"* apuntan al mismo lugar del espacio vectorial.
- **Sin namespaces (decisión consciente)** — Pinecone permite particionar en namespaces, pero la búsqueda solo mira uno a la vez. Como un paciente mezcla síntomas (*"me duele la panza y me picó algo"*), dejé todos los protocolos en un mismo namespace para que la búsqueda cruce documentos sin necesidad de un router previo. El origen de cada fragmento se distingue por **metadata** (`source`), no por namespace.

### Recuperación en runtime
El agente usa Pinecone en modo **`retrieve-as-tool`**: decide autónomamente cuándo buscar, recupera por similitud semántica, y un **reranker de Cohere** reordena los candidatos por relevancia real antes de pasárselos al modelo. Es un segundo filtro de calidad sobre el RAG.

---

## ⚖️ Evaluación automática (LLM-as-judge)

Después de cada triage, un **segundo modelo independiente** (`claude-sonnet-4.6`, `temperature: 0`) actúa como jurado y audita la respuesta. No atiende al paciente; evalúa la salida del otro sistema.

Puntúa 1–5: **completitud, coherencia, seguridad, calidad de transcripción** (solo audio), y emite una **urgencia independiente**. Esa urgencia es la clave: si el juez ve algo más grave que el triage, se detecta una **discrepancia** y el caso escala a humano.

```json
{
  "calidad_transcripcion": 4,
  "completitud": 3,
  "coherencia": 2,
  "seguridad": 3,
  "urgencia_evaluador": "MEDIA",
  "observacion": "El sistema ignoró el relato (dolor abdominal + fiebre)..."
}
```

**Caso real de las pruebas:** el agente de audio una vez ignoró el relato del paciente y clasificó como `BAJA` un caso con fiebre y dolor. El evaluador le puso coherencia 2, marcó `MEDIA`, y la discrepancia disparó el escalamiento a un médico. El sistema cazó su propio error. **Exactamente para eso sirve el juez.**

---

## 👨‍⚕️ Human-in-the-Loop (HITL)

La IA hace el 90% (recibe, transcribe, busca, redacta); el humano **aprueba o corrige** solo donde importa. Analogía: el residente atiende a todos, pero el médico de planta firma los casos delicados.

### Criterio de escalamiento
Un caso escala si se cumple **al menos uno**:
```js
debe_escalar = (urgencia === "ALTA")        // riesgo real
            || calidad_dudosa               // audio poco confiable
            || hay_discrepancia_grave        // el juez vio algo peor
```

### Mecánica
Se usa el nodo de Telegram **`sendAndWait`**, que pausa el workflow hasta que el médico responde un formulario con:
- La respuesta propuesta (editable)
- Un dropdown: aprobar / rechazar
- Nota opcional

El médico recibe un **evidence pack** (relato del paciente + triage de la IA + motivo del escalamiento) para decidir en segundos.

---

## 🔄 Fallbacks y políticas de espera

El punto que separa un HITL serio de uno de juguete: **¿qué pasa si el médico no responde?**

```
urgencia ALTA  → espera 1er médico 10 min ┐
urgencia MEDIA → espera 1er médico 30 min ├─ timeout dinámico
urgencia BAJA  → espera 1er médico 60 min ┘

  ↓ sin respuesta
2do revisor → espera la MITAD del tiempo (el paciente YA esperó)

  ↓ sin respuesta
PROTOCOLO SEGURO → avisa al paciente (deriva a guardia)
                 + alerta al equipo médico
                 (nunca queda en el limbo)
```

El segundo revisor espera la mitad **a propósito**: el tiempo ya corrió, no se puede acumular otra ventana completa en un caso urgente.

A nivel infraestructura: cada agente tiene **modelo de fallback** (si Groq falla, otro modelo toma la posta) y las llamadas críticas tienen `retryOnFail`. Ningún punto único de falla deja a un paciente sin servicio.

---

## 📊 Observabilidad y trazabilidad

Dos capas que miden cosas distintas y se complementan:

```
┌─────────────────────────┬──────────────────────────────────────┐
│ LangSmith (técnica)     │ Airtable (negocio + operación)        │
├─────────────────────────┼──────────────────────────────────────┤
│ latencia por llamada    │ CONSULTAS: 1 fila por consulta        │
│ tokens (in/out)         │   urgencias, eval scores, decisión    │
│ costo por run           │   médica, señales de audio, etc.      │
│ trazas del LLM          │ LOGS_SISTEMA: 1 fila por evento       │
│                         │   con nivel (INFO/WARNING/ERROR/CRIT) │
│ "¿cuánto tarda/cuesta?" │ "¿el triage es bueno? ¿está sano?"    │
└─────────────────────────┴──────────────────────────────────────┘
```

### El correlation ID: `id_consulta`
Cada consulta tiene un identificador único (`$execution.id`) que **viaja con ella y se escribe en cada fila de log**. Es el "número de historia clínica": con ese ID se reconstruye toda la vida de una consulta filtrando `LOGS_SISTEMA`.

```
id_consulta: 614501036
├─ INFO  consulta_recibida      Normalizar
├─ INFO  triage_generado        Urgencia detectada
├─ INFO  evaluacion_completada  Parsear
├─ WARN  discrepancia_detectada Comparacion
├─ INFO  escalado_hitl          ¿Escalamos?
├─ INFO  decision_medica        Aprobo/Rechazo
└─ INFO  respuesta_enviada      Enviar respuesta
```

Sin el ID, los logs serían eventos sueltos imposibles de conectar. **Con él, hay trazabilidad de punta a punta** — exactamente lo que NotebookLM necesita para analizar.

### Niveles de severidad
Clasificar cada evento por nivel es **aplicar triage al propio sistema**: un `CRITICO` (ej. *nadie revisó un caso escalado*) es el código rojo de la infraestructura. En producción permite filtrar "solo los CRITICO de hoy" en lugar de ahogarse en miles de líneas de INFO.

### El patrón Logger (DRY)
Loguear desde 14 puntos NO significa 14 nodos de Airtable repetidos. Se implementó un **sub-workflow `Logger`** reutilizable:

```
Flujo principal (14 checkpoints)
      │ cada uno llama vía "Execute Workflow"
      ▼
┌──────────────────────────────────┐
│  SUB-WORKFLOW: Logger             │
│  Execute Workflow Trigger          │
│    (recibe: id_consulta, nivel,   │
│     evento, rama, origen,         │
│     mensaje, detalle)             │
│        ▼                           │
│  Airtable Create → LOGS_SISTEMA   │
└──────────────────────────────────┘
```

Airtable se configura **una sola vez** (en el Logger). Cada checkpoint le pasa sus parámetros. Si mañana cambia el esquema de log, se toca en un solo lugar. Los Loggers se conectan como **ramas laterales** (no en cadena), para no romper el flujo principal pasándole la respuesta de Airtable al siguiente nodo.

Cada log guarda un campo `detalle` con `JSON.stringify($json)`: los campos indexables sirven para **filtrar**, y el blob guarda **todo el contexto** para análisis profundo. Patrón estándar: pocas columnas consultables + un payload estructurado.

---

## 🚨 Manejo de errores

Cada punto de falla queda registrado, nunca muere en silencio:

```
Agente de triage falla    → Set Error Variables → notifica + Logger ERROR (fallo_agente)
Envío a Telegram falla    → notifica → Logger ERROR (fallo_envio)
Nadie revisa caso crítico → alerta equipo → Logger CRITICO (sin_revision_critica)
```

Las ramas de error de los tres agentes convergen marcando su `origen_error` (audio/imagen/texto), de modo que el log sabe **en qué rama y etapa** ocurrió el fallo. Esto permite responder "¿qué modalidad falla más?" desde los datos.

---

## 🔁 Workflow auto-mejorable

**No es** la IA reentrenándose sola. **Es** un termostato: mide, compara con un objetivo, y reacciona si se desvía — sin supervisión manual.

```
Schedule (diario)
  → lee últimas 50 filas de CONSULTAS
  → calcula:  seguridad_promedio  +  % discrepancia
  → IF (seguridad < 3.5  OR  discrepancia > 30%)
        ├─ SÍ → alerta al equipo + Logger WARNING (auto_ajuste_calidad)
        └─ NO → Logger INFO (salud_ok)  ← deja constancia de la revisión
```

**Por qué dos métricas juntas:** detectan problemas distintos. Seguridad bajando = respuestas menos prudentes (problema de *contenido*). Discrepancia subiendo = el sistema subestima urgencias (problema de *criterio*). Son dos sensores (humo + temperatura): juntos reducen falsas alarmas.

**Por qué alerta en vez de auto-modificar:** en un contexto clínico, cambiar el comportamiento sin aprobación humana es arriesgado. La acción prudente es **detectar y avisar**; el ajuste lo confirma una persona. Coherente con el criterio de prudencia de todo el sistema.

---

## 🔒 Seguridad y privacidad

- **No se persisten identificadores del paciente en métricas.** La tabla `CONSULTAS` deliberadamente NO guarda el `chat_id` de Telegram. El dato analítico se queda; el identificable, no.
- **Disclaimer obligatorio** incrustado en el prompt de cada agente.
- **n8n self-hosted** → control total de los datos, sin pasar por terceros sin control.
- **Requisitos pre-producción documentados:** validación legal/clínica institucional, cumplimiento de Ley 25.326 (datos personales, AR), consentimiento informado explícito, y canal humano de respaldo permanente.

---

## 💡 Decisiones de diseño destacadas

| Decisión | Por qué |
|----------|---------|
| RAG en lugar de LLM solo | Un LLM alucina; en salud eso es inaceptable. Forzar grounding en protocolos reales. |
| `temperature: 0.2` en triage | Apego a la fuente y consistencia, no creatividad. |
| Quality gate de audio | No hacer triage sobre transcripciones dudosas. |
| Unificar ramas en `Normalizar` | El pipeline posterior es agnóstico al canal → escalabilidad. |
| Evaluador independiente | Segundo par de ojos automático que caza errores del agente. |
| Timeout dinámico por urgencia | El tiempo de espera tiene que escalar con el riesgo. |
| Correlation ID en logs | Trazabilidad de punta a punta. |
| Logger como sub-workflow | DRY: configurar el logging una sola vez. |
| Auto-mejora que alerta (no auto-ajusta) | Prudencia: humano en el loop también para los cambios. |

---

## ⚠️ Limitaciones conocidas

- Los umbrales del workflow auto-mejorable (3.5 / 30%) son estimaciones iniciales; deben recalibrarse con datos de producción reales.

---

## 🧰 Tech highlights 

Este proyecto demuestra, más allá de "conectar nodos":

- **Diseño de pipelines RAG** end-to-end (ingesta, chunking, embeddings, retrieval, reranking).
- **Integración multimodal** real (STT, visión, texto) con normalización a un esquema común.
- **Observabilidad de grado producción** en dos capas (técnica + negocio) con correlation IDs.
- **Patrones de resiliencia**: fallbacks de modelo, timeouts dinámicos, degradación controlada, idempotencia.
- **Evaluación automática** con LLM-as-judge y detección de discrepancias.
- **Arquitectura DRY** con sub-workflows reutilizables.
- **Monitoreo activo** con un loop de auto-mejora basado en KPIs.

---

*Proyecto desarrollado como trabajo final del curso IA Automation Avanzado (Coderhouse).*
