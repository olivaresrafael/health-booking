# Arquitectura

## Principio

Un solo motor de citas y tres adaptadores de canal. Una sola app Next.js, una sola base de datos Supabase, sin servidores adicionales.

```
 Web (reserva / gestionar) ─┐
 WhatsApp (botones/listas) ─┼──►  ENGINE (src/lib/engine)  ──►  Postgres (Supabase)
 Email (enlaces mágicos)   ─┘     reglas + máquina de estados        │
                                          │                          ▼
                                          ├──► jobs (cola en Postgres) ──► worker (/api/cron/run-jobs)
                                          └──► Google Calendar (fuente de verdad)
```

## Componentes

| Componente | Responsabilidad | Ubicación |
|---|---|---|
| Motor de citas | Reglas, estados, disponibilidad, idempotencia | `src/lib/engine` |
| Adaptadores de canal | Traducen entradas/salidas de web, WhatsApp, email al motor | `src/lib/channels`, `src/app/api/webhooks` |
| Cola de trabajos | Recordatorios, sincronización de calendario, reintentos | tabla `jobs` + `src/lib/jobs` |
| Planificador de recordatorios | Calcula qué mensajes programar por cita | `src/lib/messaging` |
| Sincronización de calendario | Lee disponibilidad, escribe eventos, procesa cambios externos | `src/lib/calendar` |
| Vertical packs | Configuración por especialidad | `src/lib/verticals` |
| Agente de IA | Interpreta texto libre y propone acciones al motor mediante herramientas | `src/lib/ai` |
| Panel | UI del profesional y asistente | `src/app/(dashboard)` |
| Páginas públicas | Reserva y gestión de cita | `src/app/(public)` |

## Flujos principales

### A. Paciente agenda por web
1. `GET /[orgSlug]` → `getAvailableSlots`.
2. Paciente elige horario → se crea un `slot_hold` de 5 minutos.
3. Envía datos + consentimiento → `bookAppointment` (revalida disponibilidad y calendario).
4. En la misma transacción: inserta cita, evento de auditoría y jobs (`calendar_upsert`, `send_message` de confirmación, recordatorios futuros).
5. El worker ejecuta los jobs.

### B. Paciente escribe por WhatsApp
1. Webhook → verificar firma → deduplicar por id de mensaje → guardar en `message_log`.
2. Identificar paciente por teléfono E.164 (o crear en el flujo de nombre).
3. Máquina conversacional (`whatsapp_sessions`) decide la respuesta; toda acción de negocio llama al motor.
4. Se responde con mensaje interactivo dentro de la ventana de 24 h (sin costo de plantilla).

### C. Recordatorios
1. Al crear o mover una cita, `planReminders` inserta jobs con `dedupe_key = <appointment_id>:<kind>`.
2. Al reagendar o cancelar, los jobs pendientes de esa cita pasan a `cancelled` y se replanifican.
3. Al ejecutarse cada job se **revalida** (estado de la cita, consentimiento, horario permitido) antes de enviar.

### D. Cambio hecho en Google Calendar
1. Google envía notificación push al webhook.
2. Se hace sincronización incremental con `syncToken`.
3. Si un evento gestionado por la app se movió o borró, se llama al motor (`channel = 'calendar'`) y se avisa al paciente.

### E. Texto libre por WhatsApp (agente de IA)
1. Webhook recibe un mensaje de texto (no botón) y aplica filtros previos: baja, emergencia, audio/imagen.
2. El `MessageRouter` (implementación base o Jev) decide la ruta: acuse fijo, paso determinista, riesgo, baja confianza o agente. Si va al agente y hay cupo mensual, `src/lib/ai` lo ejecuta con un contexto construido por el servidor (org, paciente, hora y zona horaria).
3. El agente consulta con herramientas de lectura y, si corresponde, crea una **propuesta** (`whatsapp_sessions.context.pending_proposal`).
4. El sistema envía los botones [Confirmar] / [Cancelar] con texto generado por código.
5. Al tocar [Confirmar], se ejecuta la acción en el motor con `idempotencyKey = proposalId`.
6. Cada turno registra una fila en `ai_runs`. Si el agente falla o no hay cupo, se usa el menú determinista.

Detalle en `docs/AI_AGENT.md`.

## Cola de trabajos (patrón outbox)

- Tabla `jobs`: `kind`, `run_at`, `payload`, `status`, `attempts`, `dedupe_key`.
- `pg_cron` ejecuta cada minuto una función que llama con `pg_net` a `/api/cron/run-jobs` (cabecera con secreto).
- El worker reclama trabajos con `claim_jobs(limit)` usando `FOR UPDATE SKIP LOCKED`, los ejecuta y marca resultado.
- Reintentos con espera exponencial (máximo 5). Tras agotarlos: estado `failed` y alerta.
- Si la complejidad crece, migrar a Inngest o Trigger.dev (ver ADR-003).

## Multi-tenant

- Tenant = `organizations`. Toda tabla de negocio lleva `org_id`.
- RLS por pertenencia en `members`. Las páginas públicas y los webhooks usan una clave de servicio solo en el servidor y filtran siempre por `org_id` resuelto desde el slug o el número de WhatsApp.
- Nunca exponer la clave de servicio al cliente.

## Canal: interfaz común

```ts
interface ChannelAdapter {
  readonly channel: 'whatsapp' | 'email' | 'sms';
  sendTemplate(input: {
    orgId: string;
    to: string;               // E.164 o email
    template: TemplateKey;    // ver docs/WHATSAPP_FLOWS.md
    params: Record<string, string>;
    actions?: ActionLink[];   // botones o enlaces
    idempotencyKey: string;
  }): Promise<{ providerMessageId: string }>;
}
```

El fallback (WhatsApp → email → SMS) lo decide `src/lib/messaging`, no el adaptador.

## Seguridad y privacidad

- Enlaces mágicos: token aleatorio de 32 bytes, se guarda solo el hash, vencimiento, un solo uso para acciones que cambian estado.
- CAPTCHA y límite de tasa en la reserva pública y en cualquier endpoint de enlaces.
- Datos clínicos solo en `form_submissions` (cifrado a nivel de columna para respuestas sensibles); nunca en mensajes.
- Cifrar tokens de Google antes de guardarlos.
- Auditoría: toda acción sobre una cita genera una fila en `appointment_events`.
- Cumplimiento: consentimiento registrado por canal, baja inmediata, política de retención configurable.

## Variables de entorno (mínimas)

```
NEXT_PUBLIC_SUPABASE_URL
NEXT_PUBLIC_SUPABASE_ANON_KEY
SUPABASE_SERVICE_ROLE_KEY
APP_URL
CRON_SECRET
TOKEN_ENCRYPTION_KEY
GOOGLE_CLIENT_ID
GOOGLE_CLIENT_SECRET
WHATSAPP_APP_SECRET
WHATSAPP_VERIFY_TOKEN
WHATSAPP_ACCESS_TOKEN
RESEND_API_KEY
ANTHROPIC_API_KEY
AI_MODEL
MESSAGE_ROUTER
TYPESAFE_API_KEY
SENTRY_DSN
NEXT_PUBLIC_POSTHOG_KEY
```

## Observabilidad

- Sentry para errores; PostHog para eventos de producto (reserva completada, confirmación, cancelación por canal).
- Registro estructurado sin PII. Cada job y cada webhook llevan un `correlation_id`.
- Alertas: jobs `failed`, calidad del número de WhatsApp en baja, token de Google caducado.
