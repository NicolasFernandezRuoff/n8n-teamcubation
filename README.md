# Documentación Técnica: Bot de Estudio AI (WhatsApp)

## 1. Arquitectura General
El sistema está construido sobre una arquitectura orientada a microservicios orquestada mediante n8n. Utiliza WhatsApp como interfaz de usuario y delega las tareas de procesamiento de lenguaje natural y generación de contenido a modelos de OpenAI. La persistencia de datos se maneja con MongoDB, y el almacenamiento de archivos temporales o de contexto se realiza en Google Drive.

El flujo de información centralizado recae en un orquestador principal (Microservicio 1) que recibe los webhooks de WhatsApp, clasifica la intención del usuario mediante un agente de IA y despacha la ejecución a los microservicios secundarios vía peticiones HTTP.

### Stack Tecnológico
* **Orquestación:** n8n
* **Interfaz:** WhatsApp API (Meta)
* **Inteligencia Artificial:** OpenAI (gpt-4o, gpt-4o-mini, gpt-5.2)
* **Base de Datos:** MongoDB
* **Almacenamiento:** Google Drive
* **Generación de Audio:** Google Cloud Text-to-Speech (TTS)
* **Conversión de Archivos:** ConvertAPI (HTML a PDF)

---

## 2. Detalle de Microservicios

### Microservicio 1: Orquestador y Enrutador (Main Flow)
Es el punto de entrada de la aplicación. Gestiona la recepción de mensajes de WhatsApp y determina el flujo a seguir.

* **Trigger:** WhatsApp Trigger (Webhook).
* **Procesamiento Inicial:** Extrae metadatos del usuario (nombre, número, tipo de mensaje, ID del documento si aplica). Si el mensaje contiene un documento, lo descarga usando el token y lo almacena en Google Drive.
* **Clasificador de Intenciones (AI Agent):** Analiza el texto del usuario y clasifica la intención en una de las siguientes categorías: EXAMEN, PODCAST, DISEÑO, RESPUESTA, EXPORTAR, AYUDA, INVESTIGAR, o ERROR.
* **Enrutamiento (Switch):** Dependiendo de la clasificación, invoca al microservicio correspondiente mediante peticiones HTTP (hacia los webhooks de los MS 2 al 6).
* **Formateo y Salida:** Se encarga de recibir las respuestas de los microservicios, formatear el texto (o convertir HTML a PDF para exportar) y enviar la respuesta final al usuario vía WhatsApp.

### Microservicio 2: Generación de Exámenes
Se encarga de leer un documento fuente y generar una evaluación estructurada.

* **Trigger:** Webhook (Invocado por MS1).
* **Procesamiento de Documento:** Descarga el documento desde Google Drive usando el drive_id, extrae el texto en formato PDF y lo divide en fragmentos de 3000 caracteres para no saturar el contexto de la IA (Split In Batches).
* **Generación (AI Agent):** Actúa como profesor universitario. Analiza cada fragmento y genera entre 3 y 5 preguntas (Multiple Choice, Verdadero/Falso y Desarrollo) con sus respectivas respuestas correctas y explicaciones.
* **Persistencia:** Agrupa las preguntas generadas y actualiza el registro del usuario en MongoDB.

### Microservicio 3: Corrección de Exámenes
Evalúa las respuestas enviadas por el usuario comparándolas con la base de datos.

* **Trigger:** Webhook (Invocado por MS1).
* **Recuperación de Contexto (Tool Calling):** El Agente de IA utiliza una herramienta interna que invoca a SubFlujo para buscar en MongoDB las preguntas originales y sus respuestas correctas asociadas al drive_id.
* **Evaluación (AI Agent):** Compara las respuestas del alumno con la rúbrica/respuesta correcta.
* **Salida:** Devuelve un JSON estructurado con el estado de cada pregunta (APROBADO/DESAPROBADO), un feedback pedagógico y un mensaje de resumen general.

### Microservicio 4: Generación de Podcast
Transforma un apunte en texto a un formato de audio explicativo.

* **Trigger:** Webhook (Invocado por MS1).
* **Procesamiento:** Descarga el documento de Drive y extrae el texto.
* **Guionizado (AI Agent):** Convierte el texto académico en un guion de locución dinámico, con un tono conversacional.
* **Síntesis de Voz:** Envía el guion generado a la API de Google Cloud Text-to-Speech (modelo es-US-Wavenet-B) para generar un archivo de audio MP3.

### Microservicio 5: Edición de Exámenes
Permite al usuario ajustar un examen ya generado mediante lenguaje natural (ej. "hace la pregunta 2 más difícil").

* **Trigger:** Webhook (Invocado por MS1).
* **Recuperación:** Busca las preguntas actuales en MongoDB asociadas al número de teléfono.
* **Edición (AI Agent):** Actúa como editor pedagógico. Toma las preguntas actuales y el feedback del usuario, y aplica las modificaciones (reescribir, borrar, agregar).
* **Persistencia:** Sobrescribe el array de preguntas en MongoDB con la versión actualizada.

### Microservicio 6: Investigación y Apuntes Estructurados
Genera material de estudio desde cero a partir de un tema solicitado por el usuario.

* **Trigger:** Webhook (Invocado por MS1).
* **Investigación (AI Agent):** Actúa como investigador académico. Redacta un apunte estructurado con introducción, puntos clave y conclusión sobre el tema solicitado.
* **Formateo (Code Node):** Inserta el contenido generado en una plantilla HTML predefinida con estilos CSS corporativos/académicos.
* **Conversión:** Utiliza la API de ConvertAPI para transformar el HTML en un archivo PDF listo para ser enviado al usuario.

---

## 3. Flujos Auxiliares

### SubFlujo (Consulta de Base de Datos)
Es un flujo de utilidad diseñado para ser invocado por otros flujos (específicamente utilizado como una herramienta "Tool" por el Agente de IA en el Microservicio 3).

* **Trigger:** Execute Workflow Trigger.
* **Acción:** Recibe un drive_id como parámetro de entrada, realiza un query en la colección Bot Whapp de MongoDB (archivo_drive_id) y devuelve el documento completo asociado.

---

## 4. Estructura de Datos (MongoDB)
La colección principal (Bot Whapp) centraliza el estado de la sesión del usuario. Los campos principales inferidos incluyen:
* telefono: Identificador principal del usuario (Primary Key).
* archivo_drive_id: ID del documento actual sobre el que se está trabajando.
* preguntas: Array de objetos que contiene el examen generado (tipos, opciones, respuestas correctas, feedback).
* timestamp: Registro de la última interacción o modificación.
