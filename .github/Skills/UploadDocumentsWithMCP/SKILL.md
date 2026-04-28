---
name: upload-documents-with-mcp
description: Sube documentos en formato Markdown a la Base de Conocimiento 
  usando el MCP correspondiente. Gestiona los metadatos obligatorios (cliente, 
  proyecto) y opcionales (tipo de documento). Si el tipo de documento no está 
  configurado, infiere el tipo analizando el contenido Markdown y comparándolo 
  con la lista de tipos disponibles en la base de conocimiento. Úsala siempre 
  que un documento Markdown deba ser cargado a la Base de Conocimiento.
---

# Skill — Upload Documents with MCP

Sube un documento en formato Markdown a la Base de Conocimiento usando el MCP 
disponible. Gestiona los metadatos requeridos por el sistema y, cuando el 
tipo de documento no está configurado, lo infiere automáticamente.

---

## Inputs Requeridos

| Campo              | Obligatorio | Fuente              | Descripción                                         |
|--------------------|-------------|---------------------|-----------------------------------------------------|
| `cliente`          | ✅ Sí        | `.env → CLIENTE`    | Nombre del cliente dueño del documento              |
| `proyecto`         | ✅ Sí        | `.env → PROYECTO`   | Nombre del proyecto al que pertenece                |
| `tipo_documento`   | ❌ No        | `.env → TIPO_DOCUMENTO` | Tipo de documento. Vacío = inferir automáticamente |
| `contenido_markdown` | ✅ Sí      | Skill TransformDocs | Contenido del documento en formato Markdown         |
| `nombre_archivo`   | ✅ Sí        | Agente orquestador  | Nombre original del archivo (con extensión)         |

---

## Paso 1: Leer Variables de Entorno

Lee los valores del archivo `.env` ubicado en la raíz del proyecto 
(`AGENT-INGESTORBC/.env`). Puedes usar `env.example` como referencia 
de las variables disponibles (`AGENT-INGESTORBC/env.example`).

```
CLIENTE        = [valor]
PROYECTO       = [valor]
TIPO_DOCUMENTO = [valor o vacío]
BC_API_TOKEN   = [token de autenticación]
BC_MCP_URL     = [URL del servidor MCP]
```

Si `CLIENTE` o `PROYECTO` están vacíos → **no ejecutar la carga**. 
Retornar error al agente orquestador con el mensaje:
> "Los campos CLIENTE y PROYECTO son obligatorios. Configúralos en el archivo `.env` 
> (usa `env.example` como plantilla)."

---

## Paso 2: Determinar el Tipo de Documento

### Caso A — `TIPO_DOCUMENTO` tiene valor en `.env`

Usar ese valor directamente. No realizar inferencia.

### Caso B — `TIPO_DOCUMENTO` está vacío

Ejecutar la inferencia automática usando la lista oficial de tipos de documentos 
definida en la Base de Conocimiento.

#### 2B.1 — Lista oficial de tipos de documentos

Esta es la lista canónica y definitiva de tipos disponibles en la Base de Conocimiento.
**Usar exclusivamente estos valores. No inventar ni usar sinónimos.**

| #  | Tipo de Documento                        |
|----|------------------------------------------|
| 1  | Propuestas                               |
| 2  | Información básica                       |
| 3  | Contrato                                 |
| 4  | Documentos del cliente                   |
| 5  | Iniciativa                               |
| 6  | Assesment                                |
| 7  | Arquitectura de referencia               |
| 8  | Caso de negocio                          |
| 9  | RFP                                      |
| 10 | Estimación                               |
| 11 | KickOff Externo                          |
| 12 | Políticas                                |
| 13 | Alcance                                  |
| 14 | Propuesta comercial                      |
| 15 | Acuerdos                                 |
| 16 | Portafolio de servicios                  |
| 17 | AS IS                                    |
| 18 | Repositorio documental                   |
| 19 | Riesgos - Matriz de riesgos              |
| 20 | Métricas ágiles                          |
| 21 | TO BE (estado futuro)                    |
| 22 | Diagrama de arquitectura                 |
| 23 | TO BE (estado actual)                    |
| 24 | Repositorio código fuente                |
| 25 | Documento de arquitectura                |
| 26 | Prueba de concepto                       |
| 27 | Toma de decisiones                       |
| 28 | Costos                                   |
| 29 | Integraciones                            |
| 30 | Eventos - Event Storming                 |
| 31 | Tecnologías, datos básicos               |
| 32 | Comandos - Event Storming                |
| 33 | Datos Entidades - Event Storming         |
| 34 | Usuarios - Event Storming                |
| 35 | Sistemas Externos - Event Storming       |
| 36 | Dominios - Event Storming                |
| 37 | Event Storming                           |
| 38 | Plan de configuración / DevSecOps        |
| 39 | Actas                                    |
| 40 | Riesgos - Event Storming                 |
| 41 | Story Mapping                            |
| 42 | Product Backlog                          |
| 43 | Diagrama auxiliar de requisitos          |
| 44 | Historia de usuario                      |
| 45 | Acta de cierre, encuesta satisfacción cliente |
| 46 | Diseño de pruebas                        |
| 47 | Evidencias pruebas                       |
| 48 | Plan de pruebas                          |
| 49 | Manual de configuración                  |
| 50 | Compilación                              |
| 51 | Plan CM                                  |
| 52 | Prueba unitaria                          |
| 53 | Diseño UX                                |
| 54 | Código fuente                            |
| 55 | Prueba automatizada                      |

#### 2B.2 — Analizar el contenido Markdown

Analizar el `contenido_markdown` buscando señales semánticas y cruzarlas 
contra la lista oficial:

| Señales en el contenido                                            | Tipo sugerido                          |
|--------------------------------------------------------------------|----------------------------------------|
| "propuesta", "oferta comercial", "valor económico"                 | Propuesta comercial                    |
| "contrato", "cláusula", "partes", "firma", "obligaciones"          | Contrato                               |
| "acta", "asistentes", "compromisos", "acuerdos de reunión"         | Actas                                  |
| "acta de cierre", "satisfacción", "encuesta", "cierre de proyecto" | Acta de cierre, encuesta satisfacción cliente |
| "requerimiento", "historia de usuario", "HU-", "como usuario"      | Historia de usuario                    |
| "backlog", "sprint", "épica", "prioridad", "story points"          | Product Backlog                        |
| "story map", "journey", "actividades del usuario"                  | Story Mapping                          |
| "caso de uso", "actor", "flujo alternativo", "UC-"                 | Diagrama auxiliar de requisitos        |
| "arquitectura", "diagrama", "componentes", "servicios"             | Diagrama de arquitectura               |
| "documento de arquitectura", "decisión técnica", "ADR"             | Documento de arquitectura              |
| "arquitectura de referencia", "patrón", "best practice"            | Arquitectura de referencia             |
| "estado actual", "AS IS", "situación actual"                       | AS IS                                  |
| "estado futuro", "TO BE", "situación deseada"                      | TO BE (estado futuro)                  |
| "TO BE actual", "transición", "estado de transición"               | TO BE (estado actual)                  |
| "estimación", "puntos función", "esfuerzo", "horas estimadas"      | Estimación                             |
| "costos", "presupuesto", "inversión", "CAPEX", "OPEX"              | Costos                                 |
| "riesgo", "probabilidad", "impacto", "mitigación", "matriz"        | Riesgos - Matriz de riesgos            |
| "riesgo" + "event storming"                                        | Riesgos - Event Storming               |
| "event storming", "taller", "eventos de dominio"                   | Event Storming                         |
| "comandos", "command", "event storming"                            | Comandos - Event Storming              |
| "entidades", "datos", "event storming"                             | Datos Entidades - Event Storming       |
| "usuarios", "actores", "event storming"                            | Usuarios - Event Storming              |
| "sistemas externos", "integraciones", "event storming"             | Sistemas Externos - Event Storming     |
| "dominios", "bounded context", "event storming"                    | Dominios - Event Storming              |
| "plan de pruebas", "estrategia de pruebas"                         | Plan de pruebas                        |
| "diseño de pruebas", "casos de prueba", "escenario de prueba"      | Diseño de pruebas                      |
| "evidencia", "resultado de prueba", "log de prueba"                | Evidencias pruebas                     |
| "prueba unitaria", "unit test", "cobertura"                        | Prueba unitaria                        |
| "prueba automatizada", "selenium", "cypress", "robot framework"    | Prueba automatizada                    |
| "diseño UX", "wireframe", "prototipo", "flujo de pantallas"        | Diseño UX                              |
| "código fuente", "repositorio", "rama", "commit"                   | Código fuente                          |
| "compilación", "build", "pipeline", "artefacto"                    | Compilación                            |
| "plan CM", "gestión de configuración", "baseline"                  | Plan CM                                |
| "devSecOps", "CI/CD", "configuración", "infraestructura como código"| Plan de configuración / DevSecOps     |
| "manual de configuración", "instalación", "despliegue"             | Manual de configuración                |
| "integraciones", "API", "conector", "middleware"                   | Integraciones                          |
| "tecnologías", "stack tecnológico", "datos básicos del proyecto"   | Tecnologías, datos básicos             |
| "prueba de concepto", "POC", "piloto", "demo técnica"              | Prueba de concepto                     |
| "toma de decisiones", "decisión", "alternativas evaluadas"         | Toma de decisiones                     |
| "kickoff", "arranque", "reunión de inicio"                         | KickOff Externo                        |
| "RFP", "solicitud de propuesta", "licitación"                      | RFP                                    |
| "caso de negocio", "business case", "justificación"                | Caso de negocio                        |
| "assessment", "diagnóstico", "evaluación inicial"                  | Assesment                              |
| "alcance", "scope", "límites del proyecto"                         | Acuerdos                               |
| "iniciativa", "programa", "portafolio de iniciativas"              | Iniciativa                             |
| "portafolio", "catálogo de servicios", "oferta de servicios"       | Portafolio de servicios                |
| "políticas", "lineamientos", "normativa interna"                   | Políticas                              |
| "métricas", "velocidad", "burndown", "KPI ágil"                    | Métricas ágiles                        |
| "repositorio documental", "índice de documentos"                   | Repositorio documental                 |
| Proviene de `.vtt`, `.srt` o tiene formato de hablantes            | Actas                                  |
| Proviene de `.pptx` o tiene estructura de diapositivas             | Propuesta comercial *(si aplica)*      |
| Sin coincidencia clara con ningún tipo                             | Documentos del cliente                 |

#### 2B.3 — Seleccionar el tipo inferido

- Elegir el tipo de la lista oficial (sección 2B.1) que mejor corresponda.
- Si múltiples tipos coinciden, priorizar el más específico sobre el más general.
- Si no hay coincidencia clara → usar `"Documentos del cliente"` como fallback.
- Registrar en el output: `"Tipo inferido automáticamente: [tipo seleccionado]"`.

---

## Paso 3: Ejecutar la Carga via MCP

La conexión al MCP se establece usando la configuración definida en 
`AGENT-INGESTORBC/.vscode/mcp.json`, que inyecta el token `BC_API_TOKEN` 
del `.env` en el header de autenticación.

Con todos los campos resueltos, invocar la herramienta **`crear_documento`** 
del servidor MCP `knowledge-base`:

```
knowledge-base_crear_documento({
  clientName:       "[CLIENTE]",
  projectName:      "[PROYECTO]",
  documentTypeName: "[TIPO_DOCUMENTO resuelto]",
  content:          "[contenido_markdown]",
  tags:             []
})
```

> ⚠️ **Notas sobre los parámetros:**
> - `clientName`, `projectName` y `documentTypeName` deben coincidir **exactamente** 
>   con los valores registrados en la base de conocimiento (incluyendo mayúsculas, 
>   tildes y espacios).
> - El campo `content` recibe el Markdown completo del documento.
> - El campo `tags` es opcional. Enviar array vacío `[]` si no se tienen tags adicionales.
> - **No enviar email ni usuario**: el sistema los obtiene automáticamente del token 
>   `BC_API_TOKEN` configurado en el header de autenticación.
> - El nombre del documento se genera automáticamente por el servidor MCP.

### Manejo de la respuesta del MCP

| Respuesta MCP                          | Acción                                                        |
|----------------------------------------|---------------------------------------------------------------|
| Respuesta con ID de documento generado | Registrar como cargado exitosamente con el ID retornado       |
| Error 400 (Solicitud inválida)         | Verificar que `clientName`, `projectName` y `documentTypeName` existan exactamente. Registrar error. |
| Error 401 (Unauthorized)               | Token expirado o inválido. Detener ejecución e informar al usuario para renovar `BC_API_TOKEN`. |
| Error 404 (No encontrado)              | El cliente, proyecto o tipo de documento no existe en la BC. Registrar error. |
| Timeout / sin respuesta                | Reintentar 1 vez. Si falla de nuevo → registrar error.        |
| Documento duplicado                    | Registrar advertencia, no bloquear, continuar.                |

---

## Paso 4: Retornar Resultado al Agente Orquestador

```
{
  "status": "success" | "error" | "warning",
  "nombre_archivo": "acta_reunion_2024.pdf",
  "clientName": "Empresa XYZ",
  "projectName": "Proyecto Alpha",
  "documentTypeName": "Actas",
  "tipo_inferido": true | false,
  "notas": "Tipo inferido automáticamente. Documento cargado exitosamente.",
  "document_id": "doc_abc123"
}
```

---

## Validaciones Previas a la Carga

Antes de llamar al MCP, verificar:

- [ ] `cliente` tiene valor (no vacío).
- [ ] `proyecto` tiene valor (no vacío).
- [ ] `contenido_markdown` no está vacío y tiene contenido legible.
- [ ] `tipo_documento` está resuelto (configurado o inferido).

Si alguna validación falla → retornar `status: "error"` sin invocar el MCP.

---

## Notas Importantes

- **La herramienta correcta para cargar es `crear_documento`** del servidor MCP 
  `knowledge-base`. No existe una herramienta `uploadDocument` ni equivalente; 
  usar exclusivamente `crear_documento`.
- **Nunca modificar el contenido Markdown** recibido desde la skill 
  `TransformDocsToMarkdown`. Subirlo exactamente como se recibe en el campo `content`.
- **El campo `documentTypeName`** siempre debe quedar resuelto antes de la 
  carga: ya sea desde `.env` o por inferencia. Nunca enviarlo vacío al MCP.
- **Los nombres `clientName` y `projectName`** deben coincidir exactamente con 
  los valores registrados en la base de conocimiento. Un carácter diferente 
  provocará error 400. Si hay dudas, usar `buscar_documentos` para verificar 
  la escritura exacta antes de crear.
- **La lista de tipos** está definida de forma canónica en la sección 2B.1 
  de esta skill. Usar siempre esa lista. No inventar tipos ni usar sinónimos.
- **El fallback** cuando no hay coincidencia clara es `"Documentos del cliente"`,
  nunca un valor vacío ni un tipo inventado.
- **El email del creador se obtiene automáticamente** del token `BC_API_TOKEN`. 
  No se debe pasar como parámetro a la herramienta.
- **Nunca hardcodear el token** directamente. Siempre leerlo desde 
  `AGENT-INGESTORBC/.env` mediante la variable `BC_API_TOKEN`.
