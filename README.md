# Nomunity · Checkpoint 4: integraciones con Gmail, HubSpot y Slack

Workflow de n8n para la casilla de soporte de Nomunity, que organiza viajes para nómades digitales. Cada mail que llega se filtra y se responde con IA usando la base de conocimiento de Nomunity. El remitente se registra en HubSpot sin duplicarlo, la respuesta queda como **borrador** en Gmail y el equipo recibe un aviso en Slack.

**Archivo:** `checkpoint4_silva_anapaula.json` (exportado desde n8n con Workflow → Download).

## Recorrido del flujo

```
Gmail Trigger (bandeja de entrada, cada minuto)
   │
   ▼
① ¿Es respuesta automática? ── Sí ─▶ Ignorar (corta el bucle)
   │ No
   ▼
Datos del mail (Set: email, nombre, asunto, cuerpo, id de hilo)
   │
   ├─▶ AI Agent (DeepSeek + base de conocimiento en Google Sheets)
   ▼                │
Unir mail + respuesta (Merge por posición)
   │
   ▼
④ Limpiar payload (Set) ─▶ Solo emails válidos (Filter)
   │
   ▼
② Buscar contacto en HubSpot (search por email)
   │
¿El contacto ya existe?
   ├─ Sí ─▶ Actualizar contacto
   └─ No ─▶ Crear contacto
   │
   ▼
③ Crear borrador (HITL) en Gmail
   │
   ▼
Payload mínimo para Slack (Set) ─▶ Avisar al equipo en Slack
```

## Los 4 nodos de la rúbrica

| # | Nodo en el lienzo | Qué hace | Error que previene |
|---|---|---|---|
| ① | **① ¿Es respuesta automática?** (IF) | Va inmediatamente después del trigger. Con condiciones visuales unidas por OR, descarta los mails cuyo asunto contiene *Auto-reply*, *Automatic reply*, *Respuesta automática*, *Out of office*, *Fuera de la oficina*, *Undeliverable* o *Delivery Status Notification*, o cuyo remitente contiene *no-reply*, *noreply* o *mailer-daemon*. | Bucle infinito de auto-respuestas |
| ② | **② Buscar contacto en HubSpot** + **¿El contacto ya existe?** | Busca el email en HubSpot antes de escribir. Si el contacto existe va a *Actualizar contacto*; si no, a *Crear contacto*. | Error 409 (contactos duplicados) |
| ③ | **③ Crear borrador (HITL)** (Gmail) | Usa solo la operación **Create Draft**: guarda la respuesta de la IA como borrador en el mismo hilo. Ningún mail sale sin que una persona lo revise y lo envíe. | Envíos sin aprobación humana |
| ④ | **④ Limpiar payload** (Set) + **Solo emails válidos** (Filter) | Deja solo email, nombre, asunto, cuerpo recortado, respuesta, tipo, resumen e id de hilo, sin binarios ni HTML. El filtro descarta remitentes vacíos o mal formados. | Error 400 (payload mal formado) |

Antes de Slack hay además un **Payload mínimo para Slack** (Set) que arma un único campo de texto. Así al canal nunca llegan adjuntos ni el objeto completo del mail.

## Conectores y permisos (mínimo privilegio)

| Conector | Autenticación | Lectura (pasado) | Escritura (futuro) |
|---|---|---|---|
| Gmail | OAuth2 | Mails nuevos de la bandeja de entrada | Solo borradores (Create Draft) |
| HubSpot | OAuth2 | Búsqueda de contactos por `email` (devuelve `email` y `firstname`) | Contactos: `email`, `firstname`, `lifecyclestage` (lead) y `hs_lead_status` |
| Slack | OAuth2 | — | Un mensaje de texto en el canal del equipo de operaciones |

Para HubSpot alcanza con los permisos de contactos (`crm.objects.contacts.read` y `crm.objects.contacts.write`). El agente de IA consulta la base de conocimiento en Google Sheets solo en modo lectura.

## Cómo importarlo

1. En n8n: **Workflows → Import from File** y elegí `checkpoint4_silva_anapaula.json`.
2. El archivo no incluye claves ni tokens: solo el id y el nombre de cada credencial. Asigná las tuyas en estos nodos:
   - **Gmail OAuth2:** *Gmail Trigger* y *③ Crear borrador (HITL)*.
   - **HubSpot OAuth2:** *② Buscar contacto en HubSpot*, *Actualizar contacto* y *Crear contacto*.
   - **Slack OAuth2:** *Avisar al equipo en Slack*. Elegí también tu canal.
   - **DeepSeek:** *DeepSeek Chat Model*.
   - **Google Sheets OAuth2:** las herramientas *buscar_respuestas* y *buscar_datos*. Apuntalas a tu propia base de conocimiento.

## Test de regresión

Se probó nodo por nodo y de punta a punta con mails reales:

- **Mail automático de Google** (remitente no-reply): el IF ① lo detuvo y no siguió al agente.
- **Consulta de un cliente** ("¿cuándo es el viaje a Florianópolis?"), primera ejecución: el agente respondió con la base de conocimiento, HubSpot **creó** el contacto, quedó el borrador en el hilo de Gmail y llegó el aviso a Slack.
- **Mismo mail, segunda ejecución:** HubSpot **encontró** el contacto y lo actualizó, sin duplicarlo.
