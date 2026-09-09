# n8n Workflows

Repositorio de flujos n8n exportados como `.json`, listos para importar.

## Estructura

`/flujos
  ├── clasificador-mails-circuit-breaker.json
  ├── mail-to-calendar.json
  ├── gestor-tareas-agenda.json
  └── generador-posts-redes-sociales.json
README.md

## Flujos

### 1. Clasificador de Mails con Circuit Breaker
`clasificador-mails-circuit-breaker.json`

Clasifica mails entrantes (Gmail) como `SOLICITUD_DE_REUNION` o `CONSULTA_GENERAL` usando un LLM (Groq/Llama 3.3), con fallback heurístico por palabras clave si el servicio de IA falla repetidamente.

- **Resiliencia:** circuit breaker (closed / open / half-open) persistido en Google Sheets. Ante fallos consecutivos, corta las llamadas a la IA y usa scoring por keywords.
- **Salida:** reuniones detectadas notifican por Telegram y disparan un webhook externo; consultas generales alimentan una hoja de palabras clave para mejorar el fallback heurístico.

**Servicios:** Gmail, Groq (LLM), Google Sheets, Telegram, HTTP Webhook.

### 2. Extracción de Eventos desde Mails → Google Calendar
`mail-to-calendar.json`

Detecta eventos mencionados en mails (fecha, hora, participantes) vía IA (Gemini) y los crea automáticamente en Google Calendar.

- **Deduplicación:** hash de mail (evita reprocesar el mismo correo) y hash de evento (evita crear el mismo evento dos veces).
- **Validación de fechas:** si el modelo no puede determinar fecha/hora con certeza, responde el mail pidiendo aclaración en vez de crear el evento.
- **Notificación:** confirma por Telegram la creación exitosa del evento.

**Servicios:** Gmail, Google Gemini (LLM), Google Sheets, Google Calendar, Telegram.

### 3. Sincronización de Tareas: Sheets → Calendar
`gestor-tareas-agenda.json`

Toma tareas pendientes de una hoja de Sheets, valida sus datos (fecha, estado, responsable) y agenda automáticamente en Google Calendar las que están marcadas como `lista`. Actualiza el estado en Sheets y notifica por Telegram.

- **Disparo manual, procesamiento en lote:** no reacciona a un evento externo — al ejecutarse, procesa todas las filas no marcadas como `Procesada` en una sola corrida usando `$input.all().map()`.
- **Sheets como base de datos de lectura y escritura:** única fuente de verdad y registro de estado a la vez, sin sistema externo de persistencia.

**Servicios:** Google Sheets, Google Calendar, Telegram.

### 4. Generador de Contenido Multi-Red Social
`generador-posts-redes-sociales.json`

Recibe una idea y un tono vía webhook, genera un post adaptado a la red social indicada (Instagram, LinkedIn o X) con reglas de escritura propias de cada plataforma, y guarda el resultado en Supabase.

- **Switch real:** rama por red social hacia tres prompts distintos (longitud, tono, hashtags específicos de cada plataforma).
- **Único flujo con Webhook como entrada y Supabase como persistencia** — el resto del repo usa Gmail/Manual Trigger y Sheets.

**Servicios:** Webhook, Google Gemini (LLM), Supabase.

## Cómo importar

1. En n8n: `Workflows` → `Import from File` → seleccionar el `.json`.
2. Reconfigurar credenciales (Gmail, Google Sheets, LLM, Telegram, Calendar) — los IDs de credenciales del export no son reutilizables entre instancias.
3. Actualizar IDs de spreadsheets, calendarios y `chatId` de Telegram según el entorno propio.

## Notas

- Los flujos usan Google Sheets como almacenamiento de estado (circuit breaker, hashes, palabras clave) en lugar de una base de datos dedicada — limitación intencional del entorno, no recomendada a escala.
- Ningún flujo incluye manejo de rate limits del lado de Gmail Trigger (polling cada minuto).
