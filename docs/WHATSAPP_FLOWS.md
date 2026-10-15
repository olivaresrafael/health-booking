# Flujos de WhatsApp, plantillas y email

## Convenciones

- **Opt-in:** el paciente acepta recibir mensajes en la reserva web (casilla) o al escribir primero por WhatsApp. Se registra en `consents`.
- **Baja:** "STOP", "BAJA" o "no quiero recibir mensajes" → revocar consentimiento de inmediato y confirmar con un único mensaje.
- **Ventana de 24 h:** cuando el paciente escribe o responde, se abre una ventana en la que se pueden enviar mensajes interactivos sin plantilla. Fuera de ella, solo plantillas aprobadas.
- **Categoría:** todas las plantillas son `UTILITY`. Nada de promociones ni textos de marketing.
- **Contenido:** solo logística (fecha, hora, lugar, enlaces). Nunca diagnósticos, tratamientos ni datos clínicos.
- **Tono:** configurable (`tu` por defecto, `usted` opcional).
- **Límites de mensajes interactivos** (verificar en la documentación vigente): hasta 3 botones de respuesta con título de máximo 20 caracteres; listas de hasta 10 filas con título de máximo 24 caracteres.
- **Convención de IDs de botones y filas:**

| ID | Significado |
|---|---|
| `menu:book` / `menu:change` / `menu:cancel` / `menu:human` | Opciones del menú principal |
| `slot:<ISO-UTC>:<appointmentTypeId>` | Horario elegido en una lista |
| `slot:more` | "Ver más horarios" (envía el enlace web) |
| `appt:confirm:<appointmentId>` | Confirmar asistencia |
| `appt:resched:<appointmentId>` | Cambiar la cita |
| `appt:cancel:<appointmentId>` | Cancelar la cita |
| `cancel:yes:<appointmentId>` / `cancel:no:<appointmentId>` | Confirmación final de cancelación |
| `type:<appointmentTypeId>` | Tipo de cita elegido |

## Máquina conversacional (`whatsapp_sessions.state`)

| Estado | Entra cuando | Salidas |
|---|---|---|
| `idle` | Sin conversación activa | Cualquier texto → `menu` |
| `menu` | Se mostró el menú | `menu:book` → `choosing_type` o `choosing_slot`; `menu:change` / `menu:cancel` → `choosing_appointment`; `menu:human` → `handoff` |
| `choosing_type` | Hay más de un tipo de cita | `type:*` → `choosing_slot` |
| `choosing_slot` | Se envió la lista de horarios | `slot:*` → `collecting_name` (paciente nuevo) o reserva directa; `slot:more` → enlace web |
| `collecting_name` | Paciente nuevo | Texto → reserva |
| `choosing_appointment` | El paciente tiene varias citas próximas | Selección → `confirming_cancel` o `choosing_slot` |
| `confirming_cancel` | Se pidió confirmar la cancelación | `cancel:yes` / `cancel:no` |
| `handoff` | Pidió hablar con el consultorio | Se alerta al profesional; el bot deja de responder 12 h |

Reglas:
- **Texto libre:** si el agente de IA está activo y la organización tiene cupo mensual, pasa primero por el `MessageRouter` y, si requiere conversación, lo procesa el agente (`docs/AI_AGENT.md`). Si no, o si el agente falla, se usa el respaldo por palabras clave: "cita", "agendar", "cancelar", "cambiar" o "confirmar" → paso correspondiente del menú; cualquier otro texto → menú + opción "Hablar con el consultorio".
- Botones y listas se procesan siempre de forma determinista, sin IA.
- Las propuestas del agente se confirman con botones `ai:confirm:<proposalId>` / `ai:dismiss:<proposalId>` y vencen a los 10 minutos.
- Estados con más de 30 minutos sin actividad vuelven a `idle`.
- Cualquier error del motor se traduce a un mensaje amable con salida alternativa (enlace web o contacto del consultorio).

## Flujos

### Agendar (paciente escribe primero)

> **Bot:** Hola 👋 Soy el asistente de {{profesional}}. ¿Qué necesitas?
> [📅 Agendar] [🔄 Cambiar mi cita] [❌ Cancelar mi cita]

> **Bot (lista):** Estos son los próximos horarios disponibles:
> Jue 16 oct · 4:30 pm / Vie 17 oct · 9:00 am / Vie 17 oct · 11:30 am / Lun 20 oct · 3:00 pm / Ver más horarios

> **Bot:** ¿Me dices tu nombre completo para registrar la cita?
> **Bot:** ✅ Listo, {{nombre}}. Tu cita es el {{fecha}} a las {{hora}}. Te enviaremos un recordatorio. Al continuar aceptas recibir mensajes sobre tus citas; responde BAJA para dejar de recibirlos.

### Confirmar (respuesta a la plantilla de T-24h)

`appt:confirm:<id>` → motor `confirmAppointment` → "¡Gracias, {{nombre}}! Tu cita queda confirmada ✅"

### Cambiar

`appt:resched:<id>` → lista de horarios → motor `rescheduleAppointment` → "Listo, tu nueva cita es el {{fecha}} a las {{hora}}."

### Cancelar

`appt:cancel:<id>` → "¿Seguro que quieres cancelar tu cita del {{fecha}}?" [Sí, cancelar] [No, mantenerla] → motor `cancelAppointment` → "Tu cita fue cancelada. Gracias por avisar: así otro paciente podrá usar ese horario. ¿Quieres agendar otra?" [📅 Agendar]

### Baja

"BAJA" → "Listo, no recibirás más mensajes automáticos. Si cambias de opinión, escríbenos 'cita'."

## Plantillas de WhatsApp (idioma `es`, categoría `UTILITY`)

Restricciones de parámetros: sin saltos de línea, tabuladores ni más de 4 espacios seguidos. El texto no puede empezar ni terminar con una variable. Cada plantilla se envía con ejemplos de variables para su aprobación.

Variables comunes: `{{1}}` nombre del paciente, `{{2}}` nombre del profesional, `{{3}}` fecha, `{{4}}` hora.

| Nombre | Cuándo | Cuerpo | Botones |
|---|---|---|---|
| `cita_confirmada` | Al agendar | "Hola {{1}}, tu cita con {{2}} quedó agendada para el {{3}} a las {{4}}. Dirección: {{5}}. Puedes gestionarla con el botón de abajo." | URL dinámica: "Gestionar mi cita" |
| `confirmar_24h` | T-24h | "Hola {{1}}, te recordamos tu cita con {{2}} mañana {{3}} a las {{4}}. ¿Nos confirmas tu asistencia?" | Respuesta rápida: "Confirmo" (`appt:confirm:<id>`), "Necesito cambiarla" (`appt:resched:<id>`), "Cancelar" (`appt:cancel:<id>`) |
| `confirmar_12h` | T-12h sin respuesta | "Hola {{1}}, aún no tenemos tu confirmación para tu cita de {{3}} a las {{4}} con {{2}}. Responde para reservar tu espacio." | "Confirmo", "Necesito cambiarla" |
| `instrucciones_previas` | T-48h | "Hola {{1}}, para tu cita del {{3}} con {{2}}: {{5}}" (`{{5}}` = instrucciones en una sola línea) | URL dinámica: "Completar formulario" (si aplica) |
| `formulario_previo` | Al agendar o T-72h | "Hola {{1}}, para agilizar tu consulta con {{2}} completa este formulario breve (2 minutos) antes de tu cita." | URL dinámica: "Completar formulario" |
| `logistica_1h` | T-1h | "Hola {{1}}, tu cita con {{2}} es en 1 hora ({{4}}). Dirección: {{5}}. ¡Te esperamos!" | URL dinámica: "Ver ubicación" |
| `cita_reagendada` | Tras reagendar | "Hola {{1}}, tu cita con {{2}} se movió al {{3}} a las {{4}}. Te pediremos confirmarla un día antes." | URL dinámica: "Gestionar mi cita" |
| `cita_cancelada` | Tras cancelar o si el profesional cancela | "Hola {{1}}, tu cita del {{3}} a las {{4}} con {{2}} fue cancelada. Si deseas agendar otra, responde a este mensaje." | Respuesta rápida: "Agendar nueva cita" (`menu:book`) |

## Mensajes por email (equivalentes)

Todos con asunto claro, texto breve y botones que llevan a la web con token. Remitente: nombre del profesional.

| Email | Asunto | Botones (enlaces) |
|---|---|---|
| Confirmación | "Tu cita con {{profesional}}: {{fecha}} a las {{hora}}" | Agregar al calendario (.ics), Gestionar mi cita |
| Recordatorio 24 h | "Confirma tu cita de mañana" | Confirmar, Cambiar, Cancelar |
| Instrucciones previas | "Cómo prepararte para tu consulta" | Completar formulario |
| Cancelación | "Tu cita fue cancelada" | Agendar nueva cita |

Los botones que cambian estado (Cancelar, Cambiar) abren una página que pide confirmación y ejecuta con POST.

## Alertas al profesional

| Evento | Canal | Mensaje |
|---|---|---|
| Paciente cancela | WhatsApp al profesional + panel | "{{nombre}} canceló su cita del {{fecha}} {{hora}}." |
| Sin confirmación a T-3h | WhatsApp + panel | "{{nombre}} no ha confirmado su cita de hoy a las {{hora}}." |
| Paciente pide hablar con el consultorio | WhatsApp + panel | "{{nombre}} ({{teléfono}}) quiere hablar con ustedes." |
| Fallo de entrega | Panel | "No pudimos contactar a {{nombre}}: llámalo." |
