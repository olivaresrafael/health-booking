# Integraciones

## Google Calendar

**Permisos (OAuth, mínimos):**
- `https://www.googleapis.com/auth/calendar.events`
- `https://www.googleapis.com/auth/calendar.freebusy`
- `https://www.googleapis.com/auth/calendar.calendarlist.readonly` (para elegir el calendario)

Son permisos sensibles: la app requiere **verificación de Google**. Mientras no esté verificada aparece una advertencia y hay tope de usuarios de prueba. Iniciar el trámite en la semana 0 (tarea P0-3).

**Piezas:**
- Conexión: flujo OAuth con acceso offline; guardar el refresh token cifrado (`TOKEN_ENCRYPTION_KEY`).
- Disponibilidad: `freeBusy.query` con caché corto.
- Eventos: `events.insert/patch/delete` con `extendedProperties.private.appointment_id`.
- Cambios externos: `events.watch` (notificaciones push al webhook) + `events.list` con `syncToken` para sincronización incremental. Los canales de notificación **caducan**: un job `renew_watch` los renueva antes de vencer. Si el `syncToken` se invalida (410), hacer sincronización completa.
- Manejo de errores: token revocado → marcar la conexión como `needs_reauth` y avisar al profesional.

**Después del MVP:** Outlook vía Microsoft Graph; Apple mediante feed `.ics` de solo lectura.

## WhatsApp Cloud API (Meta)

**Configuración inicial (semana 0):**
1. Crear cuenta en Meta Business Manager y completar la **verificación del negocio**.
2. Crear app de Meta con producto WhatsApp y una cuenta de WhatsApp Business (WABA).
3. Registrar un número por consultorio (el número no puede estar activo en la app de WhatsApp personal). En el MVP el alta es manual.
4. Configurar el webhook (`/api/webhooks/whatsapp`) con `WHATSAPP_VERIFY_TOKEN` y suscribir `messages`.
5. Crear y enviar a aprobación las plantillas de `docs/WHATSAPP_FLOWS.md`.

**Webhook:**
- Verificación inicial (GET con `hub.challenge`).
- Verificar `X-Hub-Signature-256` con `WHATSAPP_APP_SECRET` sobre el cuerpo sin procesar.
- Deduplicar por `messages[].id` y por `statuses[].id` + estado.
- Responder 200 rápido y procesar en segundo plano si hace falta.

**Estados de entrega:** `sent`, `delivered`, `read`, `failed`. Registrar el código de error en `message_log.error_code`.

**Errores frecuentes** (confirmar en la documentación vigente de Meta):
- Número sin WhatsApp o no entregable (p. ej. 131026) → fallback a email.
- Mensaje fuera de la ventana de 24 h sin plantilla (p. ej. 131047) → reenviar con plantilla.
- Límites de envío o de calidad (p. ej. 130429, 131049) → reducir ritmo y alertar.

**Costos (aprobación y tarifas):** la facturación es por mensaje de plantilla según categoría y país; los mensajes dentro de la ventana de servicio y las plantillas de utilidad enviadas dentro de una ventana abierta no tienen costo. Verificar la tarifa vigente aplicable a Venezuela antes de fijar precios.

**Calidad del número:** vigilar la calificación de calidad y el límite diario de envío; un volumen bajo y mensajes esperados por el paciente protegen el número.

## Email (Resend)

- Dominio propio con SPF, DKIM y DMARC.
- Plantillas con React Email.
- Webhooks de Resend (`delivered`, `bounced`, `complained`) → `message_log`; un rebote marca el email como inválido.
- Enlace de baja en cada correo.

## SMS (fase 2)

- Solo como último recurso, con créditos prepagados por organización.
- Verificar cobertura y entregabilidad en Venezuela antes de implementarlo.

## Pagos de la suscripción

- MVP: cobro manual (transferencia, Pago Móvil, Zelle, USDT) y registro del plan y límites en `organizations.plan`.
- Pasarela automática (Stripe, Lemon Squeezy, Paddle u otra) se decide con el país de la entidad que factura. Ver ADR-008.

## Jev (TypeSafe) — router de mensajes (opcional)

- **Uso:** decisiones de clasificación y enrutamiento antes del agente (ver `docs/AI_AGENT.md`, sección "Router de mensajes").
- **Acceso:** API alojada de TypeSafe; según documentación de terceros, también disponible mediante el AI Gateway de Vercel. **Confirmar en la documentación oficial** la forma de autenticación, el formato de las peticiones, los límites, el precio vigente y la política de retención de datos antes de implementar T-032.
- **Datos enviados:** solo el texto del mensaje y un estado mínimo; teléfonos y correos redactados; sin nombres.
- **Verificaciones previas:** términos de procesamiento de datos aplicables a mensajes de pacientes, rendimiento con español venezolano coloquial, latencia desde la región de despliegue.
- **Resiliencia:** tiempo máximo por llamada y respaldo automático a la implementación `baseline`.
- **Variables:** `MESSAGE_ROUTER=baseline|jev`, `TYPESAFE_API_KEY`.
