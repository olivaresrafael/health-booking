# PRD — Citas Salud (nombre provisional)

## 1. Problema

Los profesionales de la salud independientes pierden ingresos cuando los pacientes no llegan o cancelan a última hora, y pierden tiempo gestionando la agenda por WhatsApp y llamadas.

Motivos principales de inasistencia:

1. **Olvido:** la cita se agendó con días o semanas de anticipación y nadie la recordó en un canal que el paciente mire.
2. **Fricción para cancelar o reagendar:** llamar es incómodo o el consultorio está cerrado; el paciente simplemente no va.
3. **Baja urgencia o ansiedad:** el dolor mejoró, hay miedo al procedimiento o dudas sobre el costo.

## 2. Propuesta de valor

> Tus pacientes agendan, confirman o cancelan solos desde WhatsApp, email o web, y tu agenda se mantiene llena y sincronizada con tu Google Calendar, sin que cambies tu forma de trabajar.

## 3. Usuarios

| Usuario | Descripción | Necesidad principal |
|---|---|---|
| **Profesional** (owner) | Médico particular, fisioterapeuta o dentista con consultorio propio | Menos huecos en la agenda, sin aprender una herramienta nueva |
| **Asistente** (staff) | Recepcionista o secretaria (plan Clínica) | Gestionar la agenda con permisos limitados |
| **Paciente** | Persona que agenda, sin app ni contraseña | Agendar o cambiar su cita en segundos desde WhatsApp |

### Vertical packs (misma base de código)

Un pack es configuración: tipos de cita por defecto, plantillas de formulario, textos y reglas.

| Pack | Particularidades | Cuándo se activa |
|---|---|---|
| Médico | Primera consulta vs control; formulario previo de motivo de consulta | MVP |
| Dentista | Procedimientos de duración variable; instrucciones previas | MVP |
| Fisioterapeuta | **Series de sesiones** recurrentes (fase 1.5); controles frecuentes | MVP (sin series) |

## 4. Alcance del MVP

### Dentro

1. **Gestión de citas multicanal:** agendar, modificar y cancelar por web, WhatsApp y email (enlaces mágicos).
2. **Motor de secuencias automáticas:** confirmación, recordatorio con doble vía (T-24h), seguimiento, logística (T-1h).
3. **Formulario previo** simple enviado por enlace.
4. **Panel de estado** con semáforo (confirmada / sin respuesta / reagendando / cancelada) y métricas de no-shows.
5. Sincronización con **Google Calendar** (fuente de verdad).
6. Alertas al profesional cuando alguien cancela o no confirma.
7. Consentimiento y baja (STOP/BAJA).
8. **Agente de IA de citas** en WhatsApp: entiende texto libre en español, propone agendar, cambiar, cancelar y confirmar (el paciente confirma con un botón), responde datos básicos del consultorio y traspasa a una persona cuando corresponde. Ver `docs/AI_AGENT.md`.

### Fuera (fases posteriores)

Series de citas, lista de espera con relleno automático, Outlook y Apple Calendar, SMS, agente de IA para el profesional (consultar y bloquear su agenda por chat), notas de voz, WhatsApp Flows, depósitos y cobros, bandeja de conversaciones, historia clínica, app móvil.

## 5. Historias de usuario y criterios de aceptación

**H1. Paciente agenda por web**
- Dado un enlace público del profesional, el paciente ve solo horarios realmente libres (disponibilidad − citas − Google Calendar − buffers).
- Ingresa nombre, celular y acepta el consentimiento; no crea cuenta.
- Recibe confirmación por WhatsApp (o email si no hay WhatsApp) en menos de 1 minuto.
- Si dos pacientes eligen el mismo horario, uno recibe un mensaje claro y puede elegir otro.

**H2. Paciente agenda por WhatsApp**
- Al escribir al número del consultorio recibe un menú (Agendar / Cambiar mi cita / Cancelar mi cita).
- Elige entre los próximos horarios en una lista y confirma sin salir del chat.
- Si no existe en el sistema, el bot pide solo su nombre completo.

**H3. Paciente confirma**
- A T-24h recibe una plantilla con botones Confirmo / Necesito cambiarla / Cancelar.
- "Confirmo" pasa la cita a `confirmed` y se registra el evento.
- Sin respuesta a T-12h se envía un segundo aviso; a T-3h se alerta al profesional.

**H4. Paciente modifica o cancela**
- Desde WhatsApp (botones) o email/web (enlace mágico con confirmación POST).
- Se respetan las reglas del profesional: ventana de cancelación y límite de reagendados.
- El evento del calendario del profesional se actualiza.

**H5. Profesional ve su agenda**
- Panel diario y semanal con semáforo por cita y canal por el que se actuó.
- Métrica visible: citas confirmadas, cancelaciones con más de 24 h de aviso, no-shows, horas de agenda recuperadas.

**H6. Profesional edita su calendario directamente**
- Si mueve o borra un evento en Google Calendar, el paciente recibe aviso automático.

**H7. Onboarding en 15 minutos**
- Conectar Google → definir horarios y tipos de cita → conectar número de WhatsApp (asistido en el MVP) → publicar enlace.

**H8. Paciente conversa en texto libre con el agente de IA**
- El paciente escribe en lenguaje natural ("pa' mañana en la tarde") y recibe horarios reales como opciones.
- Toda acción que cambia una cita exige tocar un botón de confirmación; sin el toque no ocurre nada.
- Ante emergencias, consejos médicos o preguntas fuera de alcance, recibe un mensaje fijo y/o traspaso al consultorio.
- Si el agente falla o el consultorio agotó su cupo mensual, la conversación continúa con el menú de botones.

## 6. Métricas de éxito (producto)

| Métrica | Meta del piloto |
|---|---|
| Tasa de no-show | Reducción ≥ 40% frente a la línea base del consultorio |
| Confirmaciones por doble vía | ≥ 70% de las citas |
| Citas gestionadas por el paciente sin intervención del consultorio | ≥ 50% de cambios y cancelaciones |
| Conversaciones de texto libre resueltas por el agente sin humano | ≥ 70% |
| Acciones ejecutadas por el agente sin confirmación del paciente | 0 |
| Tiempo de onboarding | ≤ 30 minutos con asistencia |
| Costo variable por cliente | ≤ 5 USD/mes |

## 7. Contexto Venezuela (supuestos a validar)

- **WhatsApp es el canal dominante.** El email lo usan poco los pacientes: es respaldo, no canal principal. Diseñar el flujo para que funcione sin email.
- **Economía en dólares.** Precios en USD. El cobro de la suscripción en el MVP es manual (transferencia, Pago Móvil, Zelle o USDT); las tarjetas internacionales no son universales entre profesionales. Ver `docs/DECISIONS.md` (ADR-008).
- **Sensibilidad al precio.** El plan de 39 USD puede ser alto para un consultorio individual; probar 19–29 USD en la preventa.
- **Conectividad irregular.** Páginas públicas muy ligeras, tolerantes a cortes, con reintentos de envío.
- **Teléfonos.** Prefijos móviles: 0412 y 0422 (Digitel), 0414 y 0424 (Movistar), 0416 y 0426 (Movilnet). Normalizar a E.164 con `+58` y sin el 0 inicial.
- **Lenguaje del agente.** Debe entender español venezolano coloquial ("pa' mañana", "en la tarde", "a primera hora") y escribir en un tono claro y cercano.
- **Tratamiento de tú/usted.** Configurable por organización; por defecto "tú" cordial.
- **Verificaciones pendientes:** disponibilidad y términos de cada proveedor (Meta, Google, Supabase, Vercel, Resend, pasarela de pagos) para clientes en Venezuela, tarifa vigente de WhatsApp para el país y normativa local de datos de salud.
