# Project Coordinator Hub

Repositorio para usar IA como agente de coordinacion de proyectos, con respuestas rapidas y concisas, redaccion de minutas, resumenes ejecutivos y control de horas del equipo.

## Objetivo

Este repositorio te ayuda a:

- centralizar informacion del proyecto,
- pedir respuestas claras en poco tiempo,
- redactar minutas correctas,
- generar resumenes por reunion, dia o semana,
- llevar registro de horas por persona.

## Estructura

- `docs/AGENTE_COORDINADOR.md`: comportamiento recomendado del agente.
- `data/contexto-proyecto.md`: contexto general del proyecto.
- `data/personas-equipo.md`: lista de personas, roles y disponibilidad.
- `projects/`: una carpeta por proyecto (fuente principal para el agente).
- `templates/minuta-reunion.md`: plantilla de minuta.
- `templates/resumen-ejecutivo.md`: plantilla de resumen.
- `templates/seguimiento-tareas.md`: seguimiento de tareas y bloqueos.
- `templates/registro-horas.csv`: base para control de horas.
- `prompts/solicitudes-rapidas.md`: prompts listos para copiar y pegar.

## Uso rapido

1. Completa los archivos en `data/` con tu informacion real.
2. Abre `docs/AGENTE_COORDINADOR.md` y usalo como prompt base del agente.
3. Usa las plantillas de `templates/` para producir entregables consistentes.
4. Usa los prompts de `prompts/solicitudes-rapidas.md` para respuestas inmediatas.

## Carpeta por proyecto

Para cargar un proyecto nuevo:

1. Crea una carpeta dentro de `projects/` con formato:
       `CL-PRJ-XXXXX-nombre-cliente-nombre-proyecto-YYYY-MM-DD`
2. Copia dentro los insumos en `inbox/` y luego ordenalos en `fuentes/`.
3. Trabaja minutas, resumenes, tareas y horas desde esa carpeta.

Primer proyecto cargado:

- `projects/CL-PRJ-500706-gama-leasing-operativo-spa-modern-secops-2026-07-08`

## Flujo sugerido diario

1. Antes de reuniones: prepara agenda con `templates/minuta-reunion.md`.
2. Durante reuniones: registra acuerdos, responsables, fecha y riesgos.
3. Despues: pide al agente un resumen ejecutivo y siguientes pasos.
4. Fin del dia o semana: actualiza horas y seguimiento de tareas.

## Recomendacion para respuestas rapidas y concisas

Cuando hables con el agente, agrega esta linea al final de cada solicitud:

`Responde en maximo 8 lineas, con accion, responsable y fecha.`

## Resultado esperado

Con esta base tendras un asistente operativo para coordinacion de proyectos con informacion organizada, salida estandar y menor tiempo de gestion.

---

## Nuevo agente agregado: Agent-IngestorBC

Tambien se incorporo un agente para carga masiva de documentos a Base de Conocimiento.

- Definicion del agente: `.github/agents/AgentIngestorBC.agent.md`
- Variables de entorno: `.github/.env` y `.github/env.example`
- Configuracion MCP: `.vscode/mcp.json`

### Que hace

Procesa y sube documentos de multiples formatos mediante MCP:

- Office: `.pdf`, `.docx`, `.pptx`, `.xlsx` (y variantes)
- Tabulares y texto: `.csv`, `.txt`, `.html`
- Estructurados y transcripciones: `.json`, `.xml`, `.vtt`, `.srt`
- Imagenes: `.png`, `.jpg` (con OCR)

Flujo principal:

1. Lee configuracion (cliente, proyecto y tipo de documento).
2. Transforma documentos a Markdown con skills especializadas.
3. Infiere tipo de documento cuando no viene definido.
4. Sube archivos al MCP con metadatos.
5. Entrega reporte final por documento.

### Requisitos rapidos

1. Instalar skills requeridas:

```bash
npx skills add anthropics/skills@pdf
npx skills add anthropics/skills@docx
npx skills add anthropics/skills@pptx
npx skills add anthropics/skills@xlsx
```

2. Configurar `.github/.env` con:

- `CLIENTE`
- `PROYECTO`
- `BC_API_TOKEN`
- `BC_MCP_URL`
- `TIPO_DOCUMENTO` (opcional)

3. Invocar en Copilot:

```text
@agent-upload-documents-bc Sube los documentos de la carpeta Docs a la base de conocimiento
```

---