# Modelo de datos (borrador v0)

Postgres en Supabase. Todas las tablas con `org_id` tienen RLS. Las fechas son `timestamptz` (UTC). Los nombres de tablas y columnas están en inglés.

## Resumen de tablas

| Tabla | Propósito |
|---|---|
| `organizations` | Tenant: nombre, slug, zona horaria, idioma, plan, políticas (`settings`) |
| `members` | Usuarios de una organización con rol `owner` o `staff` (vinculados a `auth.users`) |
| `providers` | Profesionales que atienden pacientes; llevan `vertical` (`medico`, `fisioterapeuta`, `dentista`) |
| `appointment_types` | Tipos de cita: nombre, duración, buffer, formulario asociado |
| `availability_rules` | Horario semanal por profesional |
| `availability_exceptions` | Días bloqueados o con horario extra |
| `calendar_connections` | Conexión a Google: calendario, token cifrado, `sync_token`, canal de notificaciones |
| `patients` | Pacientes por organización; único por (`org_id`, `phone_e164`) |
| `consents` | Consentimiento por canal: concedido, revocado, origen |
| `appointments` | Citas (ver DDL) |
| `appointment_events` | Auditoría de cada acción sobre una cita |
| `jobs` | Cola de trabajos (mensajes, sincronización de calendario) |
| `message_log` | Mensajes enviados y recibidos, estados de entrega |
| `action_tokens` | Enlaces mágicos (se guarda solo el hash) |
| `slot_holds` | Retenciones temporales de horario (5 min) |
| `forms` / `form_submissions` | Formularios previos y respuestas |
| `whatsapp_numbers` | Números conectados: `phone_number_id`, `waba_id`, calidad |
| `whatsapp_sessions` | Estado conversacional por paciente, última entrada (ventana de 24 h) y `context.pending_proposal` del agente |
| `ai_runs` | Un registro por turno del agente: modelo, versión del prompt, tokens, herramientas, latencia y resultado |

Fase posterior: `appointment_series`, `waitlist_entries`, `subscriptions`.

## DDL base (borrador)

```sql
create extension if not exists btree_gist;
create extension if not exists pgcrypto;

create type appointment_status as enum
  ('pending', 'confirmed', 'cancelled', 'completed', 'no_show');

create table organizations (
  id uuid primary key default gen_random_uuid(),
  name text not null,
  slug text not null unique,
  timezone text not null default 'America/Caracas',
  locale text not null default 'es-VE',
  plan text not null default 'individual' check (plan in ('individual','clinic')),
  settings jsonb not null default '{}'::jsonb,   -- policies, tone (tu/usted), quiet hours
  created_at timestamptz not null default now()
);

create table patients (
  id uuid primary key default gen_random_uuid(),
  org_id uuid not null references organizations(id) on delete cascade,
  full_name text not null,
  phone_e164 text not null check (phone_e164 ~ '^\+[1-9][0-9]{7,14}$'),
  email text,
  preferred_channel text not null default 'whatsapp'
    check (preferred_channel in ('whatsapp','email','sms')),
  created_at timestamptz not null default now(),
  unique (org_id, phone_e164)
);

create table appointments (
  id uuid primary key default gen_random_uuid(),
  org_id uuid not null references organizations(id) on delete cascade,
  provider_id uuid not null references providers(id),
  patient_id uuid not null references patients(id),
  appointment_type_id uuid not null references appointment_types(id),
  series_id uuid,                                  -- reservado para series (fase 1.5)
  starts_at timestamptz not null,
  ends_at timestamptz not null,
  status appointment_status not null default 'pending',
  source_channel text not null
    check (source_channel in ('web','whatsapp','email','dashboard','calendar')),
  external_event_id text,                          -- id del evento en Google Calendar
  rescheduled_count int not null default 0,
  late_cancel boolean not null default false,
  confirmed_at timestamptz,
  cancelled_at timestamptz,
  cancel_reason text,
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now(),
  constraint ends_after_starts check (ends_at > starts_at),
  -- evita doble reserva a nivel de base de datos
  constraint no_overlap exclude using gist (
    provider_id with =,
    tstzrange(starts_at, ends_at, '[)') with &&
  ) where (status in ('pending','confirmed'))
);
create index appointments_org_starts_idx on appointments (org_id, starts_at);
create index appointments_patient_idx on appointments (patient_id, starts_at);

create table appointment_events (
  id uuid primary key default gen_random_uuid(),
  org_id uuid not null,
  appointment_id uuid not null references appointments(id) on delete cascade,
  type text not null,        -- booked, confirmed, rescheduled, cancelled, completed, no_show, calendar_changed
  channel text not null,
  actor text not null,       -- patient, provider, staff, system
  payload jsonb not null default '{}'::jsonb,
  idempotency_key text unique,
  created_at timestamptz not null default now()
);

create type job_status as enum ('pending','running','done','failed','cancelled','skipped');

create table jobs (
  id uuid primary key default gen_random_uuid(),
  org_id uuid not null references organizations(id) on delete cascade,
  kind text not null,        -- send_message, calendar_upsert, calendar_delete, renew_watch
  run_at timestamptz not null,
  payload jsonb not null,
  status job_status not null default 'pending',
  attempts int not null default 0,
  last_error text,
  dedupe_key text unique,
  created_at timestamptz not null default now()
);
create index jobs_due_idx on jobs (run_at) where status = 'pending';

create table action_tokens (
  id uuid primary key default gen_random_uuid(),
  org_id uuid not null,
  appointment_id uuid not null references appointments(id) on delete cascade,
  token_hash text not null unique,
  purpose text not null check (purpose in ('manage','form','cancel','reschedule')),
  expires_at timestamptz not null,
  used_at timestamptz
);
```

## Métricas del agente de IA

```sql
create table ai_runs (
  id uuid primary key default gen_random_uuid(),
  org_id uuid not null references organizations(id) on delete cascade,
  session_id uuid,
  model text not null,
  prompt_version text not null,
  route text,                                      -- decisión del router: canned, deterministic, agent, risk, low_confidence
  input_tokens int not null default 0,
  output_tokens int not null default 0,
  cached_input_tokens int not null default 0,
  tool_calls jsonb not null default '[]'::jsonb,   -- nombres de herramientas, sin PII
  latency_ms int,
  outcome text not null check (outcome in ('answered','proposal','handoff','error','capped','canned')),
  created_at timestamptz not null default now()
);
create index ai_runs_org_month_idx on ai_runs (org_id, created_at);
```

El conteo de turnos del mes por organización se calcula sobre `ai_runs` y se compara con `settings.policies.ai.monthly_turn_cap`.

`organizations.settings.practice_info` guarda lo que el agente puede responder: dirección, horarios, instrucciones previas por tipo de cita, lista de precios y preguntas frecuentes.

## Reclamar trabajos (función)

```sql
create or replace function claim_jobs(batch int)
returns setof jobs language sql as $$
  update jobs set status = 'running', attempts = attempts + 1
  where id in (
    select id from jobs
    where status = 'pending' and run_at <= now()
    order by run_at
    limit batch
    for update skip locked
  )
  returning *;
$$;
```

## Políticas de la organización (`organizations.settings.policies`)

```json
{
  "min_notice_minutes": 120,
  "cancel_window_hours": 12,
  "max_reschedules": 2,
  "new_patient_requires_approval": false,
  "slot_step_minutes": 15,
  "booking_horizon_days": 60,
  "quiet_hours": { "start": "21:00", "end": "07:00" },
  "tone": "tu",
  "ai": { "enabled": true, "monthly_turn_cap": 300 }
}
```

## RLS (patrón)

- `members` define el acceso: un usuario ve filas de las organizaciones donde es miembro.
- `staff` no puede borrar citas ni cambiar políticas ni plan.
- Las páginas públicas y los webhooks NO usan RLS de usuario: pasan por funciones del servidor con la clave de servicio, siempre acotadas a un `org_id` resuelto.
- Prueba de RLS obligatoria: un usuario de la organización A no puede leer ni escribir nada de la B.
