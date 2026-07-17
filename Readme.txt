# Sistema de Clasificación de Leads y Respuesta Automatizada con HITL

## 📄 Caso de Uso
Este proyecto resuelve la gestión y priorización eficiente de prospectos comerciales (leads) entrantes por correo electrónico. El sistema analiza el contenido del email, mitiga la duplicidad de registros y utiliza Inteligencia Artificial para categorizar el valor del lead. 

Para leads críticos o de alto valor (VIP), se implementa un circuito de **Human-In-The-Loop (HITL)** que congela la automatización hasta que un operador humano valida el borrador de respuesta. Los leads estándar o fríos se procesan y responden de forma 100% automatizada.

## 🛠️ Stack Tecnológico
*   **Orquestador de Flujos:** n8n (Self-Hosted)
*   **Procesamiento de Lenguaje Natural (LLM):** Google Gemini API
*   **Base de Datos / Persistencia:** Airtable
*   **Canal de Alertas:** Slack
*   **Proveedor de Correo:** Gmail

---

## 📐 Arquitectura y Componentes Básicos

### Workflow 1: Clasificador y Gestor de Leads
1.  **Gmail Trigger:** Captura correos entrantes filtrados por etiquetas específicas.
2.  **Validate Input:** Nodo condicional que asegura que el correo contenga cuerpo y remitente válidos.
3.  **Airtable Duplicados:** Busca el `source_message_id` para evitar re-procesamientos innecesarios.
4.  **Google Gemini (IA):** Analiza el texto, asigna un score de relevancia y redacta un borrador de respuesta (`draft_reply`).
5.  **IF (HITL Router):** Si el lead es categorizado como VIP o tiene score alto, desvía el flujo, actualiza Airtable a `Esperando Aprobación` y envía un mensaje interactivo a Slack. Si no es crítico, autodefine el flujo enviando el mail directamente.
6.  **Error Handling (Manejo de Excepciones):** Un nodo `Error Trigger` global captura fallas de API o conectividad, actualiza el estado a `Error` en Airtable y alerta al equipo vía Slack.

### Workflow 2: Approval Listener (HITL Activo)
1.  **Airtable Trigger:** Monitorea de forma síncrona mediante sondeo (polling) la columna `human_decision`.
2.  **IF Condition:** Filtra únicamente las filas donde la decisión humana sea exactamente `Aprobado`.
3.  **Gmail (Send Reply):** Despacha la respuesta final usando el contenido del borrador corregido o aprobado.
4.  **Airtable Set Executed:** Modifica el estado final del lead a `Ejecutado` cerrando el ciclo.

---

## 🚀 Guía de Ejecución

### Requisitos Previos
*   Instancia de n8n activa.
*   API Key de Google Gemini (obtenida desde Google AI Studio).
*   Cuenta de Airtable con una base estructurada (`Leads`) y las columnas de control: `status` y `last_updated` (Tipo: *Last modified time* vinculado a `human_decision`).
*   Conexiones activas en n8n para Gmail y Slack.

### Pasos para replicar
1.  Importar los archivos JSON de la carpeta `/blueprint` en tu instancia de n8n.
2.  Vincular tus credenciales correspondientes en cada nodo (Airtable, Gmail, Slack y Gemini).
3.  Configurar las variables ID de tu Base y Tabla en los nodos de Airtable de ambos flujos.
4.  Activar los interruptores (*Toggle*) de producción de ambos Workflows.

---

## 🧪 Matriz de Verificación y Pruebas

Se realizaron las 5 ejecuciones mandatorias para estresar y validar la robustez de la arquitectura:

| ID Prueba | Escenario Testeado | Comportamiento Esperado | Resultado |
| :--- | :--- | :--- | :--- |
| **01** | Lead Convencional / Frío | IA asigna score bajo. Envía correo automático de inmediato. Estado: `Procesado`. | **Exitoso** |
| **02** | Lead Corporativo / VIP | IA detecta alto valor. Frena el flujo, notifica a Slack. Espera aprobación humana. | **Exitoso** |
| **03** | Lead Incompleto | IA detecta falta de datos y genera un `draft_reply` solicitando aclaraciones. | **Exitoso** |
| **04** | Simulación de Falla Técnica | Simulación de caída de API. `Error Trigger` intercepta el fallo. Notifica a Slack y marca `Error` en BD. | **Exitoso** |
| **05** | Control de Duplicidad | Reenvío del mismo correo. El sistema detecta el ID registrado y detiene el flujo sin duplicar. | **Exitoso** |