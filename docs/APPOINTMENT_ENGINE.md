# Motor de citas

Todo cambio de una cita pasa por `src/lib/engine`. Los canales no tienen reglas propias.

## Estados

```
pending ──confirm──► confirmed
   │                    │
   ├──cancel──► cancelled ◄──cancel──┤
   │                                  │
   └────────► completed / no_show ◄───┘   (las marca el profesional o el sistema tras la hora)
```

- `reagendar` NO es un estado: es un evento. La cita mantiene su id, cambia `starts_at`/`ends_at`, incrementa `rescheduled_count` y **vuelve a `pending`** (hay que reconfirmar).
- Estados terminales: `cancelled`, `completed`, `no_show`.

| Desde | Evento | Hacia | Efectos |
|---|---|---|---|
| (nueva) | `book` | `pending` | cita + evento de auditoría + jobs de recordatorio + evento de calendario |
| `pending` | `confirm` | `confirmed` | `confirmed_at`, cancelar jobs de seguimiento |
| `pending`, `confirmed` | `reschedule` | `pending` | mover cita, `rescheduled_count + 1`, recalcular jobs, actualizar calendario |
| `pending`, `confirmed` | `cancel` | `cancelled` | `cancelled_at`, `late_cancel` si corresponde, anular jobs, borrar evento de calendario, avisar al profesional |
| `pending`, `confirmed` | `complete` | `completed` | solo después de `starts_at` |
| `pending`, `confirmed` | `mark_no_show` | `no_show` | solo después de `starts_at`; cuenta para métricas |

## Acciones del motor

```ts
type Channel = 'web' | 'whatsapp' | 'email' | 'dashboard' | 'calendar';
type Actor = 'patient' | 'provider' | 'staff' | 'system';

getAvailableSlots(i: { providerId; appointmentTypeId; from: Date; to: Date }): Promise<Slot[]>

bookAppointment(i: {
  orgId; providerId; appointmentTypeId; startsAt: Date;
  patient: { fullName: string; phone: string; email?: string };
  consent: { channel: 'whatsapp' | 'email'; source: 'web_checkbox' | 'whatsapp_optin' };
  channel: Channel; idempotencyKey: string;
}): Promise<Result<Appointment, EngineError>>

rescheduleAppointment(i: { appointmentId; newStartsAt: Date; actor: Actor; channel: Channel; idempotencyKey: string })
cancelAppointment(i: { appointmentId; reason?: string; actor: Actor; channel: Channel; idempotencyKey: string })
confirmAppointment(i: { appointmentId; actor: Actor; channel: Channel; idempotencyKey: string })
markOutcome(i: { appointmentId; outcome: 'completed' | 'no_show'; actor: Actor })
```

Todas devuelven `Result<T, EngineError>` (sin lanzar excepciones para errores de negocio).

### Errores de negocio

| Código | Cuándo |
|---|---|
| `SLOT_TAKEN` | El horario ya no está libre (restricción de solapamiento o calendario) |
| `OUTSIDE_AVAILABILITY` | Fuera del horario del profesional |
| `MIN_NOTICE` | Menos de `min_notice_minutes` de anticipación |
| `BEYOND_HORIZON` | Más allá de `booking_horizon_days` |
| `RESCHEDULE_LIMIT` | Se superó `max_reschedules` |
| `INVALID_STATE` | Transición no permitida |
| `NO_CONSENT` | El paciente no consintió ningún canal |
| `APPROVAL_REQUIRED` | Paciente nuevo cuando la política exige aprobación (queda `pending` con bandera) |

Cancelar fuera de `cancel_window_hours` **no es un error**: se permite, se marca `late_cancel = true` y se avisa al profesional.

## Idempotencia

- Cada acción lleva `idempotencyKey`. Se guarda en `appointment_events.idempotency_key` (único). Repetir la misma clave devuelve el resultado original sin duplicar efectos.
- Webhooks: la clave se deriva del id del mensaje del proveedor.

## Cálculo de horarios libres (`getAvailableSlots`)

1. Generar candidatos desde `availability_rules` en la zona horaria de la organización, en pasos de `slot_step_minutes`, para el rango pedido.
2. Aplicar `availability_exceptions` (bloqueos y horarios extra).
3. Restar citas `pending`/`confirmed` del profesional (considerando `buffer_min` del tipo de cita).
4. Restar ocupación de Google Calendar (`freeBusy`), con caché de 30–60 s.
5. Restar `slot_holds` vigentes de otras sesiones.
6. Descartar lo anterior a `now() + min_notice_minutes` y posterior a `booking_horizon_days`.
7. Devolver en UTC con la zona horaria de la organización para mostrar.

## Concurrencia

1. La selección crea un `slot_hold` (5 min).
2. `bookAppointment` revalida en la misma transacción que inserta.
3. La restricción `EXCLUDE` de `appointments` es la barrera final: si falla, se devuelve `SLOT_TAKEN`.
4. Antes de confirmar se hace una consulta `freeBusy` rápida; si el calendario cambió, `SLOT_TAKEN`.

## Planificador de recordatorios (`planReminders`)

| kind | run_at | Condición al planificar | Revalidación al enviar |
|---|---|---|---|
| `booking_confirmation` | ahora | siempre | consentimiento |
| `form_reminder` | `starts_at − 72h` | anticipación > 5 días y la cita tiene formulario | formulario sin responder |
| `prep_instructions` | `starts_at − 48h` | el tipo de cita tiene instrucciones | cita activa |
| `confirm_request` | `starts_at − 24h` | anticipación ≥ 24 h (si no, el mensaje de confirmación incluye los botones) | estado `pending` |
| `confirm_nudge` | `starts_at − 12h` | anticipación ≥ 12 h | sigue `pending` |
| `provider_alert_unconfirmed` | `starts_at − 3h` | anticipación ≥ 3 h | sigue `pending` |
| `logistics` | `starts_at − 1h` | anticipación ≥ 1 h | cita activa |

Reglas comunes:
- `dedupe_key = <appointment_id>:<kind>:<starts_at epoch>` para que reagendar genere jobs nuevos.
- **Horas de silencio** (`quiet_hours`): si un envío cae dentro, se mueve al límite permitido más cercano.
- Al cancelar o reagendar: jobs pendientes de esa cita → `cancelled`.
- Si el paciente respondió "Confirmo", `confirm_nudge` y `provider_alert_unconfirmed` quedan `skipped`.

## Fallback de canal

| Situación | Acción |
|---|---|
| Error de entrega inmediato (número sin WhatsApp) | Marcar `patients.preferred_channel = 'email'` si hay email y reenviar por email |
| Sin evento `delivered` en 30 min | Reenviar por email si existe |
| Sin email ni WhatsApp válido | Alerta al profesional para llamar |
| SMS | Fase 2, solo con créditos prepagados |

## Sincronización con el calendario

- El evento de Google lleva `extendedProperties.private.appointment_id` para reconocer eventos gestionados por la app.
- Si el profesional mueve o borra ese evento: acción del motor con `channel = 'calendar'`, `actor = 'provider'` y aviso al paciente.
- Si falla la escritura en Google: la cita existe igual; el job `calendar_upsert` reintenta y la bandera aparece en el panel.

## Pruebas obligatorias (Vitest)

- Cada transición válida e inválida de la tabla de estados.
- Solapamiento exacto, parcial y adyacente (`[)`).
- Cambios de día por zona horaria y fronteras de medianoche.
- `min_notice`, `cancel_window`, `max_reschedules`.
- Idempotencia: misma clave dos veces → un solo efecto.
- Reagendar replanifica y cancela los jobs anteriores.
