# Proceso_principal | Chatbot para Consultorio Dental — Documentación Técnica

## DOCUMENTACIÓN TÉCNICA
**Agente IA para Consultorio Dental**
---
**Plataforma:** n8n | **Canal:** WhatsApp Business | **IA:** GPT-4o-mini

## 1. Resumen Ejecutivo
Proceso_principal es un workflow de automatización construido sobre n8n que implementa un agente conversacional de inteligencia artificial llamado Lia. El agente opera como asistente virtual del consultorio dental del Dr. Luis Aceves, atendiendo a los pacientes a través de WhatsApp Business.
El agente es capaz de gestionar de forma autónoma el ciclo completo de atención: desde la bienvenida inicial hasta la confirmación de la cita, pasando por la consulta del historial del paciente, la verificación de disponibilidad en calendario, el agendamiento o modificación de citas, y la notificación al personal del consultorio vía Telegram.

| Características principales |
| --- |
| • Canal de entrada: WhatsApp Business Cloud (Meta API) <br> • Motor de IA: GPT-4o-mini (OpenAI) con memoria contextual de 10 turnos <br> • Gestión de agenda: Google Calendar (crear, leer, modificar, eliminar citas) <br> • Base de conocimiento: Google Sheets (procedimientos, info clínica, base de datos de pacientes) <br> • Notificaciones internas: Telegram Bot <br> • Zona horaria: America/Mexico_City |

## 2. Arquitectura del Flujo

### 2.1 Diagrama de flujo general

```
┌─────────────────┐     ┌──────────────────┐     ┌─────────────────┐
│   WhatsApp      │────▶│   AI Agent (Lia) │────▶│  Google Calendar│
│   Business API  │     │   (OpenAI)       │     │  (Citas)        │
└─────────────────┘     └──────────────────┘     └─────────────────┘
                               │
                               ▼
                        ┌──────────────────┐
                        │  Google Sheets   │
                        │  - Procedimientos│
                        │  - Info Clínica  │
                        │  - BD Pacientes  │
                        └──────────────────┘
                               │
                               ▼
                        ┌──────────────────┐
                        │   Telegram Bot   │
                        │   (Notificaciones)│
                        └──────────────────┘
```

### 2.2 Tabla de nodos

| Nodo | Tipo | Función |
| --- | --- | --- |
| WhatsApp Trigger | n8n-nodes-base.whatsAppTrigger | Punto de entrada. Escucha mensajes entrantes de WhatsApp Business. |
| AI Agent | @n8n/n8n-nodes-langchain.agent | Núcleo del flujo. Orquesta todas las herramientas y genera las respuestas de Lia. |
| OpenAI Chat Model | @n8n/n8n-nodes-langchain.lmChatOpenAi | Modelo de lenguaje GPT-4o-mini. Procesa el contexto y genera respuestas. |
| Simple Memory | @n8n/n8n-nodes-langchain.memoryBufferWindow | Memoria conversacional de 10 turnos por sesión (clave = array de mensajes). |
| Code | n8n-nodes-base.code | Corrige el prefijo del número mexicano (521→52) y prepara el payload de respuesta. |
| WhatsApp Business Cloud | n8n-nodes-base.whatsApp | Envía la respuesta generada por Lia al paciente. |
| WhatsApp Business Cloud1 | n8n-nodes-base.whatsApp | Ruta de error: notifica al doctor si el envío falló. |
| No Operation, do nothing | n8n-nodes-base.noOp | Terminal del flujo en caso de éxito. No realiza ninguna acción. |
| Crear_Cita | n8n-nodes-base.googleCalendarTool | Herramienta IA: crea un evento en Google Calendar. |
| Leer_Cita | n8n-nodes-base.googleCalendarTool | Herramienta IA: consulta un evento específico por ID. |
| Eliminar_Cita | n8n-nodes-base.googleCalendarTool | Herramienta IA: elimina un evento del calendario. |
| modificar_cita | n8n-nodes-base.googleCalendarTool | Herramienta IA: actualiza campos de un evento existente. |
| checar_disponibilidad | n8n-nodes-base.googleCalendarTool | Herramienta IA: lista eventos en un rango de tiempo para verificar disponibilidad. |
| Lista de procedimientos | n8n-nodes-base.googleSheetsTool | Herramienta IA: lee la hoja con precios y descripciones de tratamientos. |
| informacion_clinica | n8n-nodes-base.googleSheetsTool | Herramienta IA: lee horarios, ubicación, formas de pago y promociones. |
| base de datos lectura | n8n-nodes-base.googleSheetsTool | Herramienta IA: busca datos del paciente (última visita, procedimiento, seguro). |
| base de datos escritura | n8n-nodes-base.googleSheetsTool | Herramienta IA: registra o actualiza datos del paciente en la BD. |
| TeleBot | n8n-nodes-base.telegramTool | Herramienta IA: envía notificaciones de citas al grupo de Telegram del consultorio. |

## 3. Descripción Detallada de Nodos

### 3.1 WhatsApp Trigger
Es el disparador del flujo. Se activa cada vez que un mensaje de tipo messages llega al número de WhatsApp Business configurado. Expone toda la estructura JSON del webhook de Meta, incluyendo contacts[0].wa_id (número del remitente) y messages[0].text.body (cuerpo del mensaje).

| Campo | Valor / Descripción |
| --- | --- |
| Tipo | n8n-nodes-base.whatsAppTrigger v1 |
| Evento | messages (mensajes entrantes) |
| Credencial | YOUR_WHATSAPP_TRIGGER_CREDENTIAL_ID |
| Webhook ID | a48f413f-f7c7-4e76-bf9a-b1caaf881fae |
| Salida | Objeto JSON completo del payload de Meta WhatsApp Cloud API |

### 3.2 AI Agent
Es el corazón del sistema. Recibe el mensaje del paciente, mantiene el contexto de la conversación y decide qué herramientas invocar. El prompt del sistema define completamente la personalidad, flujo conversacional y restricciones de Lia.

| Campo | Valor / Descripción |
| --- | --- |
| Tipo | @n8n/n8n-nodes-langchain.agent v1.8 |
| Modo de prompt | define (prompt personalizado en systemMessage) |
| Input del usuario | ={{ $json.messages[0].text.body }} |
| Pasos intermedios | Activados (returnIntermediateSteps: true) |
| Herramientas | 10 herramientas conectadas (Calendar x5, Sheets x3, Telegram x1, No definidas x1) |
| Conexiones IA | OpenAI Chat Model (languageModel) + Simple Memory (memory) |

**Personalidad y comportamiento de Lia**
1. Nombre: Lia
2. Idioma: Español, adaptado al estilo del usuario
3. Estilo: Frases breves, una pregunta por mensaje, sin tecnicismos
4. Zona horaria: America/Mexico_City
5. Horario de atención configurado: Lunes a jueves, 11:00-13:00 y 17:00-19:00

**Flujo conversacional del agente**
1. Saludo inicial mencionando al Dr. Luis Aceves
2. Solicitud de nombre completo → búsqueda en base de datos
3. Identificación del motivo de consulta
4. Acción: agendar / modificar / cancelar / informar
5. Confirmación + notificación por TeleBot + registro en BD

### 3.3 OpenAI Chat Model

| Campo | Valor / Descripción |
| --- | --- |
| Tipo | @n8n/n8n-nodes-langchain.lmChatOpenAi v1.2 |
| Modelo | gpt-4o-mini |
| Temperature | 0.3 (respuestas consistentes y controladas) |
| Frequency Penalty | 0.5 (reduce repetición de frases) |
| Presence Penalty | 0.2 (favorece variedad temática sutil) |
| Credencial | YOUR_OPENAI_CREDENTIAL_ID |

### 3.4 Simple Memory
Proporciona memoria de ventana deslizante al agente. Almacena los últimos 10 intercambios de la conversación, lo que permite a Lia recordar el nombre del paciente, el motivo de consulta y las acciones realizadas dentro de la misma sesión.

| Campo | Valor / Descripción |
| --- | --- |
| Tipo | @n8n/n8n-nodes-langchain.memoryBufferWindow v1.3 |
| Tipo de clave | customKey |
| Clave de sesión | ={{ $json.messages }} (array de mensajes del trigger) |
| Ventana de contexto | 10 turnos |

> **Nota sobre la clave de sesión:** La sessionKey usa el array completo de mensajes como identificador de sesión. Esto vincula la memoria al hilo de conversación de WhatsApp, garantizando continuidad entre mensajes del mismo usuario.

### 3.5 Nodo Code — Corrección de número mexicano
México tiene una particularidad en la API de WhatsApp: algunos números se reciben con el prefijo 521 (52 + 1 de larga distancia) en lugar del correcto 52. Este nodo lo corrige antes de enviar la respuesta.

| Campo | Valor / Descripción |
| --- | --- |
| Tipo | n8n-nodes-base.code v2 |
| Lenguaje | JavaScript |
| Input | WhatsApp Trigger → contacts[0].wa_id + AI Agent → output |
| Output | { recipient: número_corregido, message: respuesta_del_agente } |
| Lógica | Si wa_id empieza con '521', se reemplaza por '52' + resto del número |

### 3.6 WhatsApp Business Cloud — Envío de respuesta
Envía el mensaje de Lia al número del paciente usando la API oficial de Meta. Está configurado con manejo de error: si el envío falla (onError: continueErrorOutput), el flujo se redirige al nodo WhatsApp Business Cloud1 para notificar al doctor.

| Campo | Valor / Descripción |
| --- | --- |
| Tipo | n8n-nodes-base.whatsApp v1 |
| Operación | send |
| Phone Number ID | YOUR_WHATSAPP_PHONE_NUMBER_ID |
| Destinatario | ={{ $json.recipient }} (viene del nodo Code) |
| Cuerpo del mensaje | ={{ $json.message }} (respuesta del AI Agent) |
| On Error | continueErrorOutput → redirige a nodo de aviso |
| Credencial | YOUR_WHATSAPP_API_CREDENTIAL_ID |

### 3.7 WhatsApp Business Cloud1 — Notificación de error
Nodo de contingencia que se activa únicamente cuando el envío principal falla. Envía un mensaje directamente al número del doctor con el texto que el paciente intentó enviar y su número de contacto.

| Campo | Valor / Descripción |
| --- | --- |
| Destinatario fijo | YOUR_DOCTOR_WHATSAPP_NUMBER |
| Mensaje | Número del paciente + texto original que no pudo procesarse |
| Activación | Solo por error output del nodo WhatsApp Business Cloud |

## 4. Herramientas del Agente IA
Todas las siguientes herramientas están conectadas al AI Agent como ai_tool. El agente las invoca autónomamente según el contexto de la conversación, sin intervención humana.

### 4.1 Herramientas de Google Calendar
Todas las herramientas de calendario apuntan al mismo calendario configurado (Pruebas Citas). Utilizan OAuth2 de Google Calendar.

**Crear_Cita**
1. Operación: create (crear evento)
2. Campos generados por IA: Start, End, Summary (nombre paciente + motivo), Description
3. Descripción para el agente: usar nombre completo del paciente en el evento

**Leer_Cita**
1. Operación: get (obtener evento por ID)
2. Campo requerido por IA: Event_ID
3. Descripción para el agente: consultar cita por nombre del paciente, retornar hora y fecha

**Eliminar_Cita**
1. Operación: delete
2. Opción: sendUpdates: all (notifica a los invitados del evento)
3. Campo requerido por IA: Event_ID
4. Descripción para el agente: verificar nombre antes de eliminar

**modificar_cita**
1. Operación: update
2. Objetivo: event
3. Campo requerido por IA: Event_ID
4. Descripción para el agente: verificar disponibilidad antes de actualizar

**checar_disponibilidad**
1. Operación: getAll (listar eventos en un rango)
2. Campos generados por IA: Return_All, After (timeMin), Before (timeMax)
3. Uso: el agente la invoca para saber si ya existe una cita antes de agendar

### 4.2 Herramientas de Google Sheets

**Lista de procedimientos**
1. Hoja: Hoja 1
2. Documento: YOUR_GOOGLE_SHEETS_PROCEDURES_URL
3. Contenido esperado: tratamientos, precios, duración estimada
4. Descripción para el agente: explicar procedimientos de forma accesible al paciente

**informacion_clinica**
1. Hoja: Hoja 1 (gid=0)
2. Documento: YOUR_GOOGLE_SHEETS_CLINIC_INFO_ID
3. Contenido esperado: horarios, ubicación, formas de pago, promociones activas
4. Descripción para el agente: responder dudas sobre el negocio

**base de datos lectura**
1. Hoja: Pacientes IA - Hoja 1
2. Documento: YOUR_GOOGLE_SHEETS_PATIENTS_DB_ID
3. Columnas esperadas: nombre, última visita, procedimiento realizado, seguro
4. Descripción para el agente: identificar si el paciente ya existe en la BD

**base de datos escritura**
1. Misma hoja que lectura (BD pacientes mejorada)
2. Operación implícita: append / update
3. Descripción para el agente: registrar nuevo paciente con 'Citado' en procedimiento, o actualizar datos tras consulta

### 4.3 TeleBot — Notificaciones a Telegram
Herramienta de notificación interna. El agente la invoca después de cada acción relevante para mantener informado al equipo del consultorio en tiempo real.

| Campo | Valor / Descripción |
| --- | --- |
| Tipo | n8n-nodes-base.telegramTool v1.2 |
| Chat ID | YOUR_TELEGRAM_CHAT_ID (grupo del consultorio) |
| Campo IA | Text — generado por el agente con nombre, fecha, hora y número del paciente |
| Credencial | YOUR_TELEGRAM_CREDENTIAL_ID |
| Cuándo | Al agendar, modificar o cancelar una cita |

> **Contenido del mensaje de Telegram:** El agente incluye automáticamente en cada notificación:
> • Nombre del paciente
> • Fecha y hora de la cita
> • Número de WhatsApp del paciente (para contactarlo si es necesario)
> • Tipo de acción realizada (nueva cita / modificación / cancelación)

## 5. Credenciales Requeridas
El workflow requiere configurar 6 credenciales en n8n antes de poder ejecutarse. Todas las credenciales deben crearse desde Settings → Credentials en la instancia de n8n. Las versiones sanitizadas del archivo usan variables YOUR_* como placeholder.

| Servicio | Tipo | Variable de entorno sugerida |
| --- | --- | --- |
| OpenAI | API Key | YOUR_OPENAI_CREDENTIAL_ID |
| WhatsApp Trigger | OAuth2 (Meta) | YOUR_WHATSAPP_TRIGGER_CREDENTIAL_ID |
| WhatsApp Business | API Key (Meta) | YOUR_WHATSAPP_API_CREDENTIAL_ID |
| Google Calendar | OAuth2 (Google) | YOUR_GOOGLE_CALENDAR_CREDENTIAL_ID |
| Google Sheets | OAuth2 (Google) | YOUR_GOOGLE_SHEETS_CREDENTIAL_ID |
| Telegram | API Token (Bot) | YOUR_TELEGRAM_CREDENTIAL_ID |

**Pasos para configurar credenciales en n8n**
1. Ir a Settings → Credentials → Add Credential
2. Seleccionar el tipo correspondiente (ej. OpenAI API, WhatsApp Business Cloud)
3. Ingresar las claves/tokens obtenidos de cada plataforma
4. Al importar el workflow, asignar cada credencial a los nodos correspondientes
5. Verificar en cada nodo que el campo 'credentials' apunte a la credencial correcta

## 6. Configuración de Recursos Externos

### 6.1 Google Sheets — Estructura de hojas requeridas

**Hoja: Lista de procedimientos**
1. Columnas sugeridas: Procedimiento | Descripción | Precio | Duración (minutos)
2. Ejemplo de fila: Limpieza dental | Profilaxis completa | $500 | 45

**Hoja: informacion_clinica**
1. Columnas sugeridas: Campo | Valor
2. Filas ejemplo: Horario | Lun-Jue 11:00-13:00 y 17:00-19:00
3. Filas ejemplo: Dirección | [dirección del consultorio]
4. Filas ejemplo: Formas de pago | Efectivo, Tarjeta, Transferencia
5. Filas ejemplo: Promoción | [texto de promoción activa o vacío]

**Hoja: BD pacientes (Pacientes IA - Hoja 1)**
1. Columnas mínimas requeridas: Nombre | Última visita | Procedimiento | Seguro
2. El agente escribe 'Citado' en Procedimiento al agendar y actualiza la fecha
3. Asegurarse de que la hoja tenga encabezados en la primera fila

### 6.2 Google Calendar
Se requiere un calendario específico creado en Google Calendar (no el calendario personal). El ID del calendario debe reemplazar YOUR_GOOGLE_CALENDAR_ID en todos los nodos de Calendar.
1. El calendario debe ser accesible por la cuenta de Google vinculada en n8n
2. Recomendado: crear un calendario dedicado llamado 'Citas Consultorio' o similar
3. No usar el calendario principal para evitar mezclar citas personales

### 6.3 WhatsApp Business (Meta)
1. Requiere cuenta de Meta Business verificada
2. Número de teléfono registrado en WhatsApp Business Platform
3. Phone Number ID debe configurarse en los nodos de envío
4. El webhook de WhatsApp debe apuntar a la URL del trigger de n8n
5. Permisos requeridos: whatsapp_business_messaging, whatsapp_business_management

### 6.4 Telegram Bot
1. Crear bot con @BotFather en Telegram
2. Obtener el token del bot y configurarlo en la credencial de Telegram en n8n
3. Agregar el bot al grupo del consultorio y obtener el Chat ID del grupo
4. El Chat ID de grupos privados tiene formato negativo: -100XXXXXXXXXX

## 7. Manejo de Errores
El flujo implementa una estrategia de error silencioso con notificación al doctor. Si el nodo principal de envío a WhatsApp falla, el flujo no se interrumpe bruscamente; en cambio, notifica al doctor con el contexto del fallo.

| Campo | Valor / Descripción |
| --- | --- |
| Nodo con error | WhatsApp Business Cloud (nodo principal de envío) |
| Configuración | onError: continueErrorOutput |
| Flujo en error | → WhatsApp Business Cloud1 (notificación al doctor) |
| Mensaje de error | Incluye: número del paciente + mensaje original |
| Flujo en éxito | → No Operation, do nothing (terminal) |

> **Limitaciones del manejo de errores actual:**
> • No existe reintentos automáticos en caso de fallo de la API de WhatsApp.
> • Los errores de herramientas IA (Google Sheets, Calendar) no tienen ruta de error explícita; el agente los maneja contextualmente.
> • No hay logging centralizado de errores más allá de la notificación por WhatsApp.

## 8. Restricciones y Reglas del Agente
El sistema prompt de Lia incluye una sección de limitaciones estrictas que el modelo debe respetar en todo momento:
1. Nunca generar citas fuera del horario establecido (Lun-Jue, 11-13 y 17-19 hrs)
2. Nunca sobreponer una cita sobre otra (verificar disponibilidad primero con checar_disponibilidad)
3. Formato de negritas: usar *texto* (un asterisco) en lugar de **texto** (dos asteriscos) — compatibilidad con WhatsApp
4. No inventar información si no la encuentra en las herramientas
5. No salir de los temas del consultorio; redirigir al usuario si se desvía
6. Al escribir el emoji de primera visita: usar 1️⃣ en el motivo de consulta

## 9. Guía de Despliegue

### 9.1 Pasos para importar y configurar
1. Importar el archivo Proceso_principal_SANITIZED.json en n8n (Workflows → Import)
2. Crear todas las credenciales descritas en la sección 5
3. Reemplazar todos los valores YOUR_* en cada nodo con los IDs/URLs reales
4. Crear y poblar las tres Google Sheets con la estructura indicada en la sección 6.1
5. Crear el calendario en Google Calendar y copiar su ID
6. Configurar el webhook de WhatsApp Business en Meta Developers apuntando al trigger de n8n
7. Activar el workflow (toggle Active en n8n)
8. Probar enviando un mensaje al número de WhatsApp Business

### 9.2 Variables a reemplazar (checklist)

| Variable placeholder | Reemplazar con |
| --- | --- |
| YOUR_OPENAI_CREDENTIAL_ID | ID de credencial OpenAI en n8n |
| YOUR_WHATSAPP_TRIGGER_CREDENTIAL_ID | ID de credencial WhatsApp Trigger en n8n |
| YOUR_WHATSAPP_API_CREDENTIAL_ID | ID de credencial WhatsApp API en n8n |
| YOUR_GOOGLE_CALENDAR_CREDENTIAL_ID | ID de credencial Google Calendar en n8n |
| YOUR_GOOGLE_SHEETS_CREDENTIAL_ID | ID de credencial Google Sheets en n8n |
| YOUR_TELEGRAM_CREDENTIAL_ID | ID de credencial Telegram en n8n |
| YOUR_WHATSAPP_PHONE_NUMBER_ID | Phone Number ID de Meta Business |
| YOUR_DOCTOR_WHATSAPP_NUMBER | Número WhatsApp del doctor (formato 52XXXXXXXXXX) |
| YOUR_TELEGRAM_CHAT_ID | Chat ID del grupo de Telegram |
| YOUR_GOOGLE_SHEETS_PROCEDURES_URL | URL de la hoja Lista de procedimientos |
| YOUR_GOOGLE_SHEETS_CLINIC_INFO_ID | ID de la hoja informacion_clinica |
| YOUR_GOOGLE_SHEETS_PATIENTS_DB_ID | ID de la hoja BD pacientes |
| YOUR_GOOGLE_CALENDAR_ID | ID del calendario de citas (@group.calendar.google.com) |
| YOUR_N8N_INSTANCE_ID | Instance ID de tu n8n (no crítico) |
| YOUR_WORKFLOW_ID | ID del workflow en n8n (se asigna al importar) |

## 10. Notas Técnicas y Consideraciones

### 10.1 Corrección del número mexicano
La API de WhatsApp Business de Meta envía algunos números mexicanos con el prefijo 521 en lugar de 52. El nodo Code detecta este patrón y lo corrige antes del envío para garantizar la entrega. Esta es una particularidad conocida de la API para números con código de larga distancia.

### 10.2 Memoria por sesión
La clave de sesión usa $json.messages (el array de mensajes del trigger). Esto significa que la memoria se mantiene mientras el contexto de la ejecución permanezca activo. Para conversaciones largas o sesiones que se retoman horas después, el agente puede perder contexto y preguntar el nombre nuevamente.

### 10.3 Formato de texto en WhatsApp
WhatsApp usa Markdown propio: *negrita* con un asterisco, _cursiva_ con guión bajo. El prompt instruye explícitamente a Lia a usar un solo asterisco (*texto*) en lugar de dos (**texto**) para garantizar compatibilidad con el formato nativo de WhatsApp.

### 10.4 Seguridad del archivo JSON
El archivo original contenía IDs de credenciales de n8n, URLs de Google Sheets, Chat IDs de Telegram, números de teléfono reales y el Instance ID de la instalación de n8n. La versión sanitizada (SANITIZED) reemplazó todos estos valores con placeholders YOUR_* para su publicación segura en repositorios públicos.

---
