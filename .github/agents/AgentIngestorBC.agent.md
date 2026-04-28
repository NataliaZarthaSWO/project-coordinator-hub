---
name: agent-upload-documents-bc
description: Agente orquestador que procesa múltiples documentos en diferentes 
  formatos (.pdf, .docx, .doc, .csv, .ppt, .pptx, transcripciones de Teams, 
  imágenes, entre otros) y los sube a la Base de Conocimiento. Delega la 
  transformación a Markdown a los skills especializados de Anthropic (pdf, docx, 
  pptx, xlsx) y gestiona los metadatos (cliente, proyecto, tipo de documento) 
  desde un archivo de variables de entorno. Úsalo cuando necesites cargar grandes 
  volúmenes de documentos a la base de conocimiento de forma masiva y parametrizada.
---

# Agente - Upload Documents to Base de Conocimiento

Agente orquestador encargado de procesar y subir documentos a la Base de 
Conocimiento. Coordina dos skills especializadas: transformación a Markdown y 
carga mediante MCP.

---

## Flujo General del Agente

```
[Leer .env] → [Validar parámetros] → [Por cada documento:]
                                          ↓
                                   [¿Ya es Markdown?]
                                    /             \
                                  Sí               No
                                  ↓                 ↓
                             [Usar tal cual]  [TransformDocsToMarkdown]
                                   \               /
                                    ↓             ↓
                              [UploadDocumentsWithMCP]
                                       ↓
                              [Reporte final de carga]
```

---

## Paso 1: Leer Configuración

Antes de procesar cualquier documento, lee los siguientes archivos ubicados 
en el proyecto. **Todos los comandos se ejecutan desde la raíz del proyecto 
`AGENT-INGESTORBC/`.**

Leer el archivo `.env`:
```bash
cat .github/.env
```

Estructura de archivos de configuración:

```
AGENT-INGESTORBC/
├── .github/
│   ├── .env            ← Variables de negocio y token MCP
│   └── env.example     ← Plantilla de referencia para las variables de entorno
└── .vscode/
    └── mcp.json        ← Configuración de conexión al MCP
```

### Variables del archivo `.env`

| Variable        | Obligatorio | Descripción                                           |
|-----------------|-------------|-------------------------------------------------------|
| `CLIENTE`       | ✅ Sí        | Nombre del cliente dueño de los documentos            |
| `PROYECTO`      | ✅ Sí        | Nombre del proyecto al que pertenecen                 |
| `TIPO_DOCUMENTO`| ❌ No        | Tipo de documento. Si está vacío, se infiere          |
| `BC_API_TOKEN`  | ✅ Sí        | Bearer token de autenticación para el MCP             |
| `BC_MCP_URL`    | ✅ Sí        | URL del servidor MCP de la Base de Conocimiento       |

**Si `CLIENTE`, `PROYECTO`, `BC_API_TOKEN` o `BC_MCP_URL` están vacíos → detener ejecución e informar al usuario.**

### Configuración del archivo `mcp.json`

El archivo `mcp.json` define la conexión al MCP de la Base de Conocimiento.
El token `BC_API_TOKEN` del `.env` se inyecta en el header `Authorization`.

```json
{
  "mcpServers": {
    "knowledge-base": {
      "url": "${BC_MCP_URL}",
      "type": "http",
      "headers": {
        "Authorization": "Bearer ${BC_API_TOKEN}"
      }
    }
  }
}
```

> ⚠️ **Nunca hardcodear el token directamente en `mcp.json`.**  
> Siempre leerlo desde `.env` mediante la variable `BC_API_TOKEN`.

---

## Paso 2: Inventariar los Documentos

> ⚠️ **IMPORTANTE — Ejecución real obligatoria:** El agente debe usar sus 
> herramientas de terminal para ejecutar comandos reales. No simular ni 
> asumir resultados. Cada paso debe producir una salida visible de herramienta.

Identifica todos los documentos a procesar ejecutando:

```bash
# Listar todos los archivos en la carpeta Docs con su extensión
find Docs/ -type f | sort
```

Los documentos están en la carpeta `Docs/` en la raíz del proyecto.
Construye un listado con: nombre del archivo, formato detectado, ruta completa.

---

## Paso 3: Procesar Cada Documento

> ⚠️ **Para cada documento, el agente DEBE ejecutar los siguientes comandos 
> reales en terminal. No describir lo que haría — ejecutarlo.**

### 3.1 — Determinar si requiere transformación

| Condición                            | Acción                                    |
|--------------------------------------|-------------------------------------------|
| El archivo ya es `.md`               | Usar directamente, saltar al Paso 3.3     |
| El archivo es cualquier otro formato | Delegar al skill correspondiente          |

### 3.2 — Transformar a Markdown usando Skills de Anthropic

> ⚠️ **IMPORTANTE:** No usar scripts Python. Delegar a los skills de Anthropic.

> ✅ **INSTALACIÓN AUTOMÁTICA:** Antes de invocar un skill, el agente debe 
> verificar si está instalado. Si no está disponible, instalarlo 
> automáticamente usando `npx skills add anthropics/skills@<skill-name>`.

**Estrategia de verificación e instalación:**

```python
# Pseudocódigo para cada documento

# 1. Detectar formato
file_extension = Path(file_path).suffix.lower()

# 2. Mapear formato a skill
skill_map = {
    ".pdf": "pdf",
    ".docx": "docx",
    ".doc": "docx",
    ".pptx": "pptx",
    ".ppt": "pptx",
    ".xlsx": "xlsx",
    ".xls": "xlsx"
}

if file_extension in skill_map:
    skill_name = skill_map[file_extension]
    
    # 3. Verificar si el skill está instalado
    skill_path = f"~/.agents/skills/{skill_name}/SKILL.md"
    
    if not path_exists(skill_path):
        # 4. Instalar el skill automáticamente
        run_command(f"npx skills add anthropics/skills@{skill_name}")
        print(f"✅ Skill '{skill_name}' instalado correctamente")
    
    # 5. Invocar el skill
    markdown_content = invoke_skill(skill_name, file_path)
else:
    # Formato sin skill especializado → usar método directo
    markdown_content = process_with_direct_method(file_path)
```

**Estrategia por formato:**

| Formato              | Skill Anthropic | Verificación antes de invocar                           |
|---------------------|-----------------|--------------------------------------------------------|
| `.pdf`              | `pdf`           | Verificar instalación → Si no existe, instalar → Invocar |
| `.docx` / `.doc`    | `docx`          | Verificar instalación → Si no existe, instalar → Invocar |
| `.pptx` / `.ppt`    | `pptx`          | Verificar instalación → Si no existe, instalar → Invocar |
| `.xlsx` / `.xls`    | `xlsx`          | Verificar instalación → Si no existe, instalar → Invocar |
| `.csv`              | N/A             | Directo: pandas → `df.to_markdown()`                    |
| `.txt`              | N/A             | Directo: Lectura directa del archivo                    |
| `.html`             | N/A             | Directo: BeautifulSoup → `get_text()`                   |
| `.json` / `.xml`    | N/A             | Directo: Pretty print como code block                   |

**Comandos de instalación por skill:**

```bash
# Si el skill PDF no está instalado
npx skills add anthropics/skills@pdf

# Si el skill DOCX no está instalado
npx skills add anthropics/skills@docx

# Si el skill PPTX no está instalado
npx skills add anthropics/skills@pptx

# Si el skill XLSX no está instalado
npx skills add anthropics/skills@xlsx
```

**Verificación de instalación:**

```bash
# Verificar si un skill está instalado listando el directorio
ls -l ~/.agents/skills/pdf/SKILL.md        # Para PDF
ls -l ~/.agents/skills/docx/SKILL.md       # Para DOCX
ls -l ~/.agents/skills/pptx/SKILL.md       # Para PPTX
ls -l ~/.agents/skills/xlsx/SKILL.md       # Para XLSX

# Si el archivo existe → skill instalado
# Si no existe → instalar con npx skills add
```

**Ejemplo de flujo completo para un PDF:**

```
1. Detectar formato: archivo.pdf → extensión .pdf
2. Mapear a skill: .pdf → skill "pdf"
3. Verificar instalación:
   - Ejecutar: ls ~/.agents/skills/pdf/SKILL.md
   - Si error "No such file" → Instalar:
     npx skills add anthropics/skills@pdf
   - Si existe → Continuar
4. Invocar skill 'pdf' con el archivo
5. Recibir markdown limpio
6. Continuar al paso 3.3 (Determinar tipo de documento)
```

**Ejemplo de flujo completo para un DOCX:**

```
1. Detectar formato: informe.docx → extensión .docx
2. Mapear a skill: .docx → skill "docx"
3. Verificar instalación:
   - Ejecutar: ls ~/.agents/skills/docx/SKILL.md
   - Si error → Instalar: npx skills add anthropics/skills@docx
   - Si existe → Continuar
4. Invocar skill 'docx' con el archivo
5. Recibir markdown limpio
6. Continuar al paso 3.3
```

📄 Ver documentación completa en:
`.github/Skills/TransformDocsToMarkdown/SKILL.md`

### 3.3 — Determinar el Tipo de Documento

| Condición                              | Acción                                          |
|----------------------------------------|-------------------------------------------------|
| `TIPO_DOCUMENTO` tiene valor en `.env` | Usar ese valor directamente                     |
| `TIPO_DOCUMENTO` está vacío            | Inferir el tipo analizando el contenido Markdown |

La inferencia del tipo se delega a la skill `UploadDocumentsWithMCP`.

### 3.4 — Subir a la Base de Conocimiento

Invoca la skill `UploadDocumentsWithMCP` pasando:
- Contenido Markdown del documento
- `CLIENTE` (del .env)
- `PROYECTO` (del .env)
- `TIPO_DOCUMENTO` (del .env o vacío para inferencia)
- Nombre original del archivo

📄 Ver instrucciones completas en:
`.github/Skills/UploadDocumentsWithMCP/SKILL.md`

---

## Paso 4: Generar Reporte Final

Al terminar de procesar todos los documentos, presenta un reporte con la 
siguiente estructura:

```
## Resumen de Carga - Base de Conocimiento

Cliente  : [CLIENTE]
Proyecto : [PROYECTO]
Fecha    : [fecha y hora]

| # | Documento              | Tipo Inferido / Configurado | Estado     | Notas              |
|---|------------------------|-----------------------------|------------|--------------------|
| 1 | acta_reunion.pdf       | Acta de Reunión (inferido)  | ✅ Cargado  |                    |
| 2 | propuesta_v2.docx      | Propuesta Comercial (env)   | ✅ Cargado  |                    |
| 3 | diagrama_arq.png       | Manual Técnico (inferido)   | ✅ Cargado  | Imágenes omitidas  |
| 4 | datos_cliente.xlsx     | —                           | ❌ Error    | Ver detalle abajo  |

Total procesados : 4
Exitosos         : 3
Con errores      : 1
```

Para cada documento con error, incluye el detalle del fallo y una sugerencia 
de acción correctiva.

---

## Manejo de Errores

| Situación                              | Comportamiento                                              |
|----------------------------------------|-------------------------------------------------------------|
| `.env` no encontrado                   | Detener y pedir al usuario que configure el archivo `.env`  |
| `CLIENTE` o `PROYECTO` vacíos          | Detener y solicitar los valores obligatorios                |
| Documento corrupto o ilegible          | Registrar error, continuar con el siguiente documento       |
| Fallo en transformación a Markdown     | Registrar error, continuar con el siguiente documento       |
| Fallo en carga al MCP                  | Registrar error, continuar con el siguiente documento       |
| Formato de archivo no soportado        | Registrar como omitido, continuar con el siguiente          |

El agente **nunca debe detenerse** por el fallo de un documento individual.
Siempre debe continuar procesando el resto y reportar al final.

---

## Consideraciones Importantes

- **Las imágenes dentro de documentos no se cargan** a la base de conocimiento.
  La transformación a Markdown debe omitirlas por completo (no incluir 
  referencias a imágenes, ni placeholders).
- **Documentos que son solo imágenes** (`.png`, `.jpg`, etc.) se transforman
  extrayendo únicamente el texto legible (OCR si aplica).
- **El orden de procesamiento** sigue el orden del inventario construido en 
  el Paso 2.
- **Cada documento se sube de forma independiente**; un fallo no bloquea 
  los demás.
