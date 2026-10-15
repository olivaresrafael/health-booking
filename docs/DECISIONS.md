# Decisiones de arquitectura (ADR)

Formato: contexto → decisión → consecuencias. Estado: `aceptada`, `propuesta` o `reemplazada`.

## ADR-001 — Una sola app Next.js, sin backend separado
- **Estado:** aceptada
- **Contexto:** equipo pequeño, 30 días de plazo, webhooks y tareas cortas.
- **Decisión:** páginas, panel, API y webhooks en una app Next.js sobre Vercel.
- **Consecuencias:** menos infraestructura. Si aparecen procesos largos, se evalúa un worker aparte.

## ADR-002 — Supabase como base y autenticación
- **Estado:** aceptada
- **Contexto:** multi-tenant con aislamiento por organización.
- **Decisión:** Postgres + RLS + Auth + Storage de Supabase, migraciones SQL versionadas y tipos generados.
- **Consecuencias:** RLS obligatoria y probada; las rutas públicas usan clave de servicio acotada por `org_id`.

## ADR-003 — Cola de trabajos propia sobre Postgres (outbox)
- **Estado:** aceptada
- **Contexto:** recordatorios con hora exacta, cancelables al reagendar.
- **Decisión:** tabla `jobs` + `claim_jobs` + `pg_cron`/`pg_net` que invoca un worker en la app.
- **Consecuencias:** sin servicio extra y fácil de probar. Migrar a Inngest o Trigger.dev si hace falta flujos con esperas largas o paralelismo.

## ADR-004 — WhatsApp Cloud API directa
- **Estado:** aceptada
- **Contexto:** minimizar costo por mensaje; pocos clientes al inicio.
- **Decisión:** Cloud API de Meta con una WABA propia y un número por consultorio, alta manual en el MVP.
- **Consecuencias:** más trabajo de configuración que un BSP, sin recargo por mensaje. Embedded Signup cuando haya volumen.

## ADR-005 — Sin IA en el MVP
- **Estado:** reemplazada por ADR-011
- **Contexto original:** menús y botones cubrían la mayoría de casos.
- **Decisión original:** agente de texto libre en fase 2.
- **Por qué cambió:** el agente de IA es prioridad del producto para el MVP.

## ADR-006 — Google Calendar como fuente de verdad
- **Estado:** aceptada
- **Decisión:** no construir calendario propio; leer disponibilidad y escribir eventos con `extendedProperties`.
- **Consecuencias:** requiere verificación de Google y sincronización incremental.

## ADR-007 — Tres verticales mediante "vertical packs"
- **Estado:** aceptada
- **Contexto:** se lanza con médico, fisioterapeuta y dentista a la vez.
- **Decisión:** un solo producto; los packs son configuración (tipos de cita, formularios, textos, reglas). El mensaje comercial se enfoca en una vertical a la vez.
- **Consecuencias:** riesgo de dispersión; mitigado con packs sin código propio. Las series de sesiones (fisioterapia) quedan en fase 1.5.

## ADR-008 — Cobro de la suscripción en el MVP
- **Estado:** propuesta (pendiente: país de la entidad que factura)
- **Contexto:** mercado inicial Venezuela; las pasarelas con tarjeta tienen cobertura limitada.
- **Decisión provisional:** cobro manual (transferencia, Pago Móvil, Zelle, USDT) con registro del plan en `organizations.plan`.
- **Consecuencias:** sin automatización de cobro al inicio; revisar tras la validación.

## ADR-009 — Reagendar es un evento, no un estado
- **Estado:** aceptada
- **Decisión:** la cita conserva su id y vuelve a `pending` tras reagendar; se reconfirma.
- **Consecuencias:** historial completo en `appointment_events`; jobs de recordatorio se replanifican.

## ADR-010 — WhatsApp como canal principal, email como respaldo
- **Estado:** aceptada
- **Contexto:** en Venezuela el email es poco usado por los pacientes.
- **Decisión:** el flujo debe funcionar completo solo con WhatsApp; el email complementa y sirve de fallback.
- **Consecuencias:** el fallback por email cubre solo a quienes lo proporcionen; el panel alerta al profesional cuando no hay canal válido.

## ADR-011 — Agente de IA de citas en el MVP, con motor determinista debajo
- **Estado:** aceptada
- **Contexto:** el agente de IA es parte esencial del MVP; los riesgos son costo, errores de interpretación y manejo de datos de salud.
- **Decisión:** agente con *tool calling* (Vercel AI SDK + Claude) que solo **propone** acciones; la ejecución ocurre en el motor tras un botón de confirmación. Botones y listas siguen sin IA. Contexto construido por el servidor. Tope mensual de turnos por organización y degradación al menú determinista.
- **Consecuencias:** +1 semana de desarrollo y una suite de evaluación obligatoria. El agente no puede causar cambios sin consentimiento explícito, y el sistema funciona aunque el modelo falle o se agote el cupo.

## ADR-012 — Router de mensajes con implementación base y Jev opcional
- **Estado:** propuesta (se adopta Jev solo si supera a la base en la evaluación)
- **Contexto:** muchos mensajes de texto libre no requieren generación (acuses, saludos, respuestas simples) y se necesita una segunda capa de seguridad para urgencias y consejo médico. Jev es un modelo de decisión de TypeSafe, muy reciente, con evidencia pública mayormente de terceros y sin verificación en español venezolano.
- **Decisión:** definir la interfaz `MessageRouter` con una implementación `baseline` (MVP) y otra `jev` activable por `MESSAGE_ROUTER`. Los filtros deterministas siguen primero y la confirmación con botón no se omite. El atajo de confirmación directa solo aplica con P ≥ 0.95 y una única cita pendiente.
- **Consecuencias:** 1–2 días extra para la interfaz y la implementación base; la integración con Jev (T-032) es opcional y reversible. Ahorro esperado modesto en dólares (a medir) y mejoras de latencia y seguridad. Nuevo proveedor que procesa texto de pacientes: requiere revisión de términos.
