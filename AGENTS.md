# AGENTS.md — Citas Salud (nombre provisional)

> Contexto principal para opencode. Léelo completo antes de tocar código. El detalle vive en `docs/`.

## Qué es este proyecto

Micro SaaS B2B para profesionales de la salud independientes (médicos particulares, fisioterapeutas y dentistas) que reduce las inasistencias (no-shows) automatizando el ciclo completo de la cita por WhatsApp, Email y web.

El paciente puede **agendar, confirmar, modificar y cancelar** desde cualquiera de los tres canales, sin app y sin contraseña. La agenda del profesional (Google Calendar) es la **fuente de verdad**: no construimos un calendario propio.

- Producto independiente y nuevo.
- Mercado inicial: **Venezuela**.
- Verticales desde el inicio: médico, fisioterapeuta, dentista, mediante "vertical packs" (configuración, no código distinto). Ver `docs/ARCHITECTURE.md`.
- Fase actual: **pre-código y piloto concierge**. Ver `docs/ROADMAP.md`.
- Idioma: documentación y UI en español (es-VE). Código, identificadores, commits y comentarios en inglés.

## Stack

- Next.js (App Router) + TypeScript estricto, desplegado en Vercel. Una sola app, sin backend separado.
- Supabase: Postgres, Auth, Row Level Security, Storage, `pg_cron` + `pg_net`.
- Tailwind + shadcn/ui, react-hook-form, zod.
- WhatsApp Cloud API de Meta (directa, sin BSP). Email con Resend + React Email.
- Google Calendar API (Outlook y SMS en fases posteriores).
- Agente de IA: Vercel AI SDK + Claude (API de Anthropic) con *tool calling* contra el motor de citas. Ver `docs/AI_AGENT.md`.
- Vitest (unitarias del motor) + Playwright (flujo de reserva). Sentry + PostHog.
- Gestor de paquetes: pnpm.

## Estructura objetivo del repositorio

```
src/
  app/
    (public)/[orgSlug]/          # reserva pública
    (public)/c/[token]/          # gestionar cita (enlace mágico)
    (dashboard)/                 # panel del profesional
    api/webhooks/{whatsapp,google-calendar,resend}/route.ts
    api/cron/run-jobs/route.ts   # lo invoca pg_cron
  lib/
    engine/                      # motor de citas: dominio + acciones
    channels/                    # adaptadores: whatsapp, email (sms después)
    calendar/                    # google (outlook después)
    messaging/                   # plantillas, planificador de recordatorios
    jobs/                        # cola de trabajos (tabla jobs)
    ai/                          # agente de IA: router de mensajes, prompts, herramientas, guardarraíles, evals
    verticals/                   # packs: medico, fisioterapeuta, dentista
    db/                          # cliente supabase, tipos generados
    utils/                       # normalizePhone, tz, tokens
  components/
supabase/{migrations,seed.sql}
tests/{unit,e2e}
docs/
.opencode/{agent,command}
```

## Comandos (se confirman al hacer el scaffold, tarea T-001)

```
pnpm install
pnpm dev
pnpm build
pnpm lint
pnpm typecheck
pnpm test            # vitest
pnpm test:e2e        # playwright
pnpm db:start        # supabase start
pnpm db:reset        # supabase db reset
pnpm db:types        # genera tipos TS desde el esquema
```

## Reglas no negociables

1. **Toda mutación de citas pasa por `src/lib/engine`.** Los adaptadores de canal (web, WhatsApp, email) nunca escriben en `appointments` directamente.
2. **Idempotencia:** cada acción recibe una `idempotency_key`; los webhooks se deduplican por el id del proveedor.
3. **RLS en todas las tablas con `org_id`.** No se acepta una migración sin políticas RLS.
4. **Fechas en UTC** (`timestamptz`). La zona horaria vive en la organización (por defecto `America/Caracas`, UTC-4, sin horario de verano).
5. **Teléfonos en E.164** (+58…). Un único helper `normalizePhone` convierte formatos venezolanos (`0414-1234567` → `+584141234567`).
6. **Nunca enviar datos clínicos** por WhatsApp ni email: solo logística y enlaces.
7. **Consentimiento primero:** no se envía ningún mensaje sin registro en `consents`. Se respeta STOP/BAJA de inmediato.
8. **Todas las plantillas de WhatsApp son categoría UTILITY.** Nada promocional.
9. **Validar con zod** toda entrada externa (formularios, API, webhooks) y verificar la firma de los webhooks.
10. **Secretos solo en variables de entorno.** No registrar PII en logs (enmascarar teléfono y nombre).
11. **El agente de IA nunca escribe en la base de datos.** Solo propone acciones; las ejecuta `src/lib/engine` tras un botón de confirmación del paciente. Botones y listas se procesan siempre sin IA. Sin teléfonos, emails ni datos clínicos en los prompts. El `MessageRouter` decide si un texto libre necesita al agente; los filtros deterministas (BAJA, emergencias) nunca dependen solo de él. Ver `docs/AI_AGENT.md`.
12. **Enlaces de email que cambian estado** (cancelar, reagendar): el GET solo muestra una página; la acción se ejecuta con POST tras confirmación del paciente (los antivirus precargan enlaces).
13. **No añadir servicios ni dependencias grandes sin un ADR** en `docs/DECISIONS.md`.

## Cómo trabajar en este repo

1. Antes de implementar, lee el documento de `docs/` relacionado con la tarea y la tarea en `docs/ROADMAP.md`.
2. Trabaja en pasos pequeños: esquema → motor → pruebas → adaptador de canal → UI.
3. Escribe pruebas para cualquier regla del motor (`docs/APPOINTMENT_ENGINE.md`) antes de conectar canales.
4. Después de cambios: `pnpm typecheck && pnpm lint && pnpm test`.
5. Si una decisión contradice un documento, **actualiza el documento y `docs/DECISIONS.md`** en el mismo cambio.
6. Si algo es ambiguo o requiere una decisión de producto, pregunta antes de inventar.
7. No toques `supabase/migrations` ya aplicadas: crea una migración nueva.

## Índice de documentos

| Documento | Contenido |
|---|---|
| `docs/PRD.md` | Producto, usuarios, alcance del MVP, historias y criterios de aceptación, contexto Venezuela |
| `docs/ARCHITECTURE.md` | Componentes, flujos, multi-tenant, cola de trabajos, seguridad |
| `docs/DATA_MODEL.md` | Tablas, DDL base, RLS |
| `docs/APPOINTMENT_ENGINE.md` | Máquina de estados, reglas, acciones, cálculo de horarios |
| `docs/AI_AGENT.md` | Agente de IA: herramientas, guardarraíles, costos, evaluación |
| `docs/WHATSAPP_FLOWS.md` | Conversaciones, botones, plantillas de WhatsApp y email |
| `docs/INTEGRATIONS.md` | Google Calendar, WhatsApp Cloud API, Resend |
| `docs/ROADMAP.md` | Plan de 30 días con tareas ejecutables |
| `docs/VALIDATION_PLAN.md` | Entrevistas, piloto concierge, precios, Go/No-Go |
| `docs/DECISIONS.md` | Decisiones de arquitectura (ADR) |
