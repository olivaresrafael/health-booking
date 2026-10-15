# Roadmap de 30 días

Marca `[x]` cuando una tarea termine. Usa `/implement-task T-XXX` en opencode. Las tareas P0 son de gestión externa y corren en paralelo desde el día 1 porque tienen tiempos de espera ajenos al código.

## Semana 0 — Gestión externa (en paralelo)

- [ ] **P0-1** Registrar dominio y configurar email transaccional (SPF, DKIM, DMARC).
- [ ] **P0-2** Meta: Business Manager, verificación del negocio, app de WhatsApp, WABA y número de prueba.
- [ ] **P0-3** Google Cloud: proyecto, pantalla de consentimiento, permisos mínimos (`docs/INTEGRATIONS.md`) y **solicitud de verificación**.
- [ ] **P0-4** Enviar a aprobación las plantillas de `docs/WHATSAPP_FLOWS.md`.
- [ ] **P0-5** Piloto concierge con 3 consultorios (uno por vertical). Ver `docs/VALIDATION_PLAN.md`.
- [ ] **P0-6** 10 entrevistas, landing y preventa. Ver `docs/VALIDATION_PLAN.md`.
- [ ] **P0-7** Verificar términos y disponibilidad en Venezuela de cada proveedor y la tarifa vigente de WhatsApp.

## Semana 1 — Base y motor

- [ ] **T-001** Scaffold: Next.js + TypeScript estricto + Tailwind + shadcn/ui + ESLint + Vitest + Playwright + CI + pnpm. *Aceptación:* `pnpm dev`, `typecheck`, `lint`, `test` pasan; los comandos de `AGENTS.md` quedan confirmados.
- [ ] **T-002** Supabase local y migraciones base (`organizations`, `members`, `providers`, `appointment_types`, `availability_*`, `patients`, `consents`) con RLS y pruebas de aislamiento. *Dep:* T-001.
- [ ] **T-003** Auth del profesional y onboarding: organización, profesional, horarios, tipos de cita según el vertical pack. *Dep:* T-002.
- [ ] **T-004** Vertical packs (`medico`, `fisioterapeuta`, `dentista`) como configuración tipada. *Dep:* T-003.
- [ ] **T-005** Motor: `getAvailableSlots` con pruebas (zonas horarias, buffers, excepciones). *Dep:* T-002.
- [ ] **T-006** Migración de `appointments`, `appointment_events` y restricción `EXCLUDE`; motor: `book`, `confirm`, `reschedule`, `cancel`, `markOutcome` con idempotencia y pruebas de la tabla de estados. *Dep:* T-005.

## Semana 2 — Web, calendario y cola

- [ ] **T-007** Cola de trabajos: tabla `jobs`, `claim_jobs`, `/api/cron/run-jobs`, `pg_cron` + `pg_net`, reintentos. *Dep:* T-002.
- [ ] **T-008** Google OAuth + `calendar_upsert`/`calendar_delete` + `freeBusy` integrado a `getAvailableSlots`. *Dep:* T-006, T-007.
- [ ] **T-009** Notificaciones push de Google + sincronización incremental + job `renew_watch`. *Dep:* T-008.
- [ ] **T-010** Página pública de reserva `/[orgSlug]` con `slot_holds`, CAPTCHA y consentimiento. *Dep:* T-006.
- [ ] **T-011** Página "Gestionar mi cita" `/c/[token]` con enlaces mágicos (GET muestra, POST ejecuta). *Dep:* T-006.
- [ ] **T-012** Email transaccional (Resend): confirmación, `.ics`, webhooks de rebote. *Dep:* T-007.

## Semana 3 — WhatsApp y recordatorios

- [ ] **T-013** Webhook de WhatsApp: verificación, firma, deduplicación, `message_log`, estados de entrega. *Dep:* T-002.
- [ ] **T-014** `ChannelAdapter` de WhatsApp y email: envío de plantillas con parámetros. *Dep:* T-012, T-013.
- [ ] **T-015** `planReminders` + jobs con revalidación al enviar y horas de silencio. *Dep:* T-007, T-014.
- [ ] **T-016** Menú conversacional (agendar, cambiar, cancelar, confirmar, hablar con el consultorio, BAJA). *Dep:* T-013, T-006.
- [ ] **T-017** Fallback a email y alertas al profesional. *Dep:* T-015.
- [ ] **T-018** Consentimiento (opt-in y baja) en todos los canales. *Dep:* T-013.

## Agente de IA (parte del MVP, semanas 3–4)

Construir **después** del menú determinista (T-016), que es su respaldo.

- [ ] **T-026** Agente base en `src/lib/ai`: Vercel AI SDK + Claude, prompt de sistema versionado, resolución de fechas relativas con la zona horaria de la organización, límite de 6 pasos y de tokens. *Dep:* T-006.
- [ ] **T-027** Herramientas (lectura, propuestas, traspaso, baja) y flujo propuesta → botón de confirmación → motor, con `idempotencyKey = proposalId`. *Dep:* T-026, T-016.
- [ ] **T-028** Guardarraíles: filtro de emergencias con mensaje fijo, rechazo de consejo médico, defensa ante inyección de prompt, aislamiento de datos entre pacientes. *Dep:* T-027.
- [ ] **T-029** Presupuesto y métricas: tabla `ai_runs`, tope mensual por organización, degradación al menú, indicadores en el panel. *Dep:* T-027.
- [ ] **T-030** Evaluación: 50 conversaciones en español de Venezuela, ejecutor en Vitest con respuestas grabadas y script en vivo; cumplir los criterios de `docs/AI_AGENT.md`. *Dep:* T-028.
- [ ] **T-031** `MessageRouter`: interfaz + implementación `baseline` (palabras clave, acuses, intenciones simples; lo dudoso va al agente), integrada en el webhook antes del agente; registrar `route` en `ai_runs`. *Dep:* T-026, T-016.
- [ ] **T-032** *(opcional)* Implementación `jev` del router + evaluación comparativa y regla de adopción de `docs/AI_AGENT.md`. Confirmar antes API vigente, límites, precio y términos de datos del proveedor. *Dep:* T-030, T-031.

## Semana 4 — Panel, formularios y piloto

- [ ] **T-019** Formularios previos: constructor simple, enlace mágico, respuestas cifradas. *Dep:* T-011.
- [ ] **T-020** Panel con semáforo, filtros y métricas (no-shows, cancelaciones con más de 24 h, horas recuperadas). *Dep:* T-006.
- [ ] **T-021** Reporte mensual de ROI por email. *Dep:* T-020, T-012.
- [ ] **T-022** Planes y límites por organización (conteo de citas, excedente) con cobro manual. *Dep:* T-020.
- [ ] **T-023** Seguridad: límites de tasa, auditoría de RLS, revisión con `/review`, Sentry, PostHog. *Dep:* todas.
- [ ] **T-024** Pruebas e2e del flujo de reserva, confirmación y cancelación. *Dep:* T-016.
- [ ] **T-025** Despliegue en producción y arranque del piloto con los consultorios. *Dep:* T-024.

## Fase 1.5 y 2 (no empezar antes de validar)

Series de sesiones (fisioterapia), lista de espera con relleno automático, Outlook, SMS, agente de IA con texto libre, WhatsApp Flows, solicitud de reseñas, reactivación de pacientes, depósitos.

## Orden de recorte si falta tiempo

El agente de IA añade aproximadamente una semana. **T-032 (Jev) es opcional y se difiere primero** si falta tiempo. Para mantener los 30 días, recorta además en este orden:

1. Reporte mensual de ROI (T-021).
2. Formularios (T-019).
3. Planes y límites (T-022): contar a mano.
4. Métricas avanzadas del panel (T-020): dejar solo semáforo y no-shows.

**No se recortan:** la web de reserva (T-010/T-011), los recordatorios (T-015), el menú determinista (T-016, es el respaldo del agente) y el agente con sus guardarraíles y presupuesto (T-026 a T-029). De T-030 se puede reducir el conjunto a 20 conversaciones, nunca omitir las de emergencia, inyección y datos ajenos.
