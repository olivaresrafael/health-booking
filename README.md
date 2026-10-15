# Citas Salud (nombre provisional)

Micro SaaS para profesionales de la salud independientes (médicos, fisioterapeutas y dentistas) que reduce los no-shows: el paciente agenda, confirma, cambia o cancela por WhatsApp, email o web, con un agente de IA que entiende texto libre. Mercado inicial: Venezuela.

Este repositorio está en fase **pre-código**: contiene el contexto para trabajar con [opencode](https://opencode.ai).

## Cómo arrancar

1. `git init` y primer commit con estos archivos.
2. Copia `.env.example` a `.env.local` cuando llegues a la tarea que lo necesite.
3. Abre opencode en esta carpeta: carga `AGENTS.md` y los documentos listados en `opencode.json`.
4. Empieza por la **Semana 0** de `docs/ROADMAP.md` (trámites con tiempos de espera: Meta, Google, plantillas) y en paralelo ejecuta `/implement-task T-001`.
5. Avanza en el orden del roadmap; usa `/review` antes de cada commit importante.

## Comandos de opencode incluidos

| Comando | Para qué |
|---|---|
| `/implement-task T-005` | Implementa una tarea del roadmap con plan, pruebas y actualización del estado |
| `/new-migration <descripción>` | Crea una migración de Supabase con RLS y regenera tipos |
| `/review` | Revisión de solo lectura de los cambios actuales |

Subagentes: `db-architect`, `whatsapp-integrator`, `ai-agent-engineer`, `qa-reviewer`.

## Mapa de documentos

| Archivo | Contenido |
|---|---|
| `AGENTS.md` | Contexto principal y reglas no negociables |
| `docs/PRD.md` | Producto, alcance, historias, contexto Venezuela |
| `docs/ARCHITECTURE.md` | Componentes y flujos |
| `docs/DATA_MODEL.md` | Esquema y RLS |
| `docs/APPOINTMENT_ENGINE.md` | Estados, reglas y acciones |
| `docs/AI_AGENT.md` | Agente de IA, router de mensajes, guardarraíles, evaluación |
| `docs/WHATSAPP_FLOWS.md` | Conversaciones, plantillas y emails |
| `docs/INTEGRATIONS.md` | Google, WhatsApp, Resend, Jev |
| `docs/ROADMAP.md` | Plan de 30 días con tareas T-001 a T-032 |
| `docs/VALIDATION_PLAN.md` | Piloto, entrevistas, precios, Go/No-Go |
| `docs/DECISIONS.md` | Decisiones de arquitectura (ADR-001 a ADR-012) |

## Supuestos actuales (confírmalos o cámbialos en los documentos)

- Documentación en español; código, identificadores y commits en inglés.
- Desarrollo en solitario con opencode.
- Cobro de la suscripción manual en el MVP (ADR-008, pendiente: país de la entidad que factura).
- Las cifras de costos de WhatsApp y del modelo de IA son estimaciones: verifica las tarifas vigentes.
- Jev (router de mensajes) es opcional y se adopta solo si supera a la implementación base (ADR-012).

## Verificaciones pendientes antes de lanzar

- Términos y disponibilidad de cada proveedor (Meta, Google, Supabase, Vercel, Resend, proveedor del modelo, TypeSafe) para clientes en Venezuela y datos de salud.
- Tarifa vigente de WhatsApp aplicable a Venezuela.
- Verificación del negocio en Meta y verificación de la app en Google (tardan días o semanas).
- Marco legal local sobre datos de salud.
