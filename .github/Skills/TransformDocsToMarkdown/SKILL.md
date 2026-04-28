---
name: transform-docs-to-markdown
description: >
  Convierte documentos de múltiples formatos (.pdf, .docx, .doc, .pptx, .ppt,
  .xlsx, .xls, .csv, .txt, .rtf, .vtt, .srt, .html, .json, .xml, imágenes)
  a Markdown limpio para su carga en la Base de Conocimiento.
  Delega a los skills especializados de Anthropic (pdf, docx, pptx, xlsx),
  instalándolos automáticamente si no están disponibles.
version: "3.0"
---

# Skill — transform-docs-to-markdown

Convierte documentos a Markdown limpio delegando a skills especializados de Anthropic.
Esta skill es invocada por el agente `AgentIngestorBC` para cada archivo no-Markdown
encontrado en `Docs/`.

## ⚙️ Arquitectura v3.0

**Cambio principal:** Delegación a skills de Anthropic con instalación automática.

**Beneficios:**
- ✅ Sin scripts Python complejos que mantener
- ✅ Skills especializados probados por la comunidad
- ✅ Actualizaciones automáticas cuando Anthropic mejora los skills
- ✅ Instalación automática para equipos de trabajo


---

## 📋 Delegación por formato

| Formato | Delegado a | Fallback/Método directo |
|---------|------------|------------------------|
| `.pdf` | **Skill `pdf`** (PyPDF + markitdown) | markitdown |
| `.docx` / `.doc` | **Skill `docx`** (python-docx + pandoc) | pandoc |
| `.pptx` / `.ppt` | **Skill `pptx`** (python-pptx + markitdown) | markitdown |
| `.xlsx` / `.xls` | **Skill `xlsx`** (openpyxl + pandas) | pandas |
| `.csv` | Procesamiento directo con pandas | — |
| `.txt` | Lectura directa | — |
| `.rtf` | Lectura directa con post-procesamiento | — |
| `.vtt` / `.srt` | Procesamiento con regex | — |
| `.html` / `.htm` | BeautifulSoup4 | — |
| `.json` | json stdlib | — |
| `.xml` | xml.etree stdlib | — |
| Imágenes | pytesseract + Pillow (OCR) | — |

**Skills de Anthropic usados:**
- `anthropics/skills@pdf` — [https://skills.sh/anthropics/skills/pdf](https://skills.sh/anthropics/skills/pdf)
- `anthropics/skills@docx` — [https://skills.sh/anthropics/skills/docx](https://skills.sh/anthropics/skills/docx)
- `anthropics/skills@pptx` — [https://skills.sh/anthropics/skills/pptx](https://skills.sh/anthropics/skills/pptx)
- `anthropics/skills@xlsx` — [https://skills.sh/anthropics/skills/xlsx](https://skills.sh/anthropics/skills/xlsx)

---

## 🔧 Verificación e instalación automática de skills

**IMPORTANTE:** Antes de usar un skill de Anthropic, debes verificar que esté instalado.
Si no está disponible, instálalo automáticamente.

### Workflow de verificación

```python
# 1. Verificar si el skill está instalado
skill_name = "pdf"  # o "docx", "pptx", "xlsx"
skill_path = Path.home() / ".agents" / "skills" / skill_name / "SKILL.md"

if not skill_path.exists():
    # 2. Instalar el skill automáticamente
    install_skill(skill_name)

# 3. Invocar el skill
invoke_skill(skill_name, input_file)
```

### Función de instalación

```bash
# Instalar un skill de Anthropic
npx skills add anthropics/skills@{skill_name}
```

### Ejemplo completo

```python
def ensure_skill_installed(skill_name: str) -> bool:
    """
    Verifica si un skill está instalado. Si no, lo instala automáticamente.
    
    Args:
        skill_name: "pdf", "docx", "pptx", o "xlsx"
    
    Returns:
        True si el skill está disponible (ya instalado o recién instalado)
    """
    skill_path = Path.home() / ".agents" / "skills" / skill_name / "SKILL.md"
    
    if skill_path.exists():
        return True
    
    # Instalar el skill
    import subprocess
    result = subprocess.run(
        ["npx", "skills", "add", f"anthropics/skills@{skill_name}"],
        capture_output=True,
        text=True,
        timeout=60
    )
    
    if result.returncode == 0:
        print(f"✅ Skill '{skill_name}' instalado correctamente")
        return True
    else:
        print(f"❌ Error instalando skill '{skill_name}': {result.stderr}")
        return False
```

---

## 📌 Instrucciones para el agente

Cuando recibas un documento para convertir a Markdown:

### Paso 1: Detectar el formato

```python
extension = Path(file_path).suffix.lower()
```

### Paso 2: Delegación según formato

#### Para PDF (.pdf)

```python
# 1. Verificar/instalar skill
if not ensure_skill_installed("pdf"):
    # Fallback: usar markitdown directamente
    return convert_pdf_with_markitdown(file_path)

# 2. Invocar el skill "pdf"
# El skill retorna el contenido markdown directamente
markdown_content = invoke_skill("pdf", file_path)

# 3. Retornar el markdown
return markdown_content
```

#### Para DOCX (.docx, .doc)

```python
# 1. Verificar/instalar skill
if not ensure_skill_installed("docx"):
    # Fallback: usar pandoc directamente
    return convert_docx_with_pandoc(file_path)

# 2. Invocar el skill "docx"
markdown_content = invoke_skill("docx", file_path)

# 3. Retornar el markdown
return markdown_content
```

#### Para PPTX (.pptx, .ppt)

```python
# 1. Verificar/instalar skill
if not ensure_skill_installed("pptx"):
    # Fallback: usar markitdown directamente
    return convert_pptx_with_markitdown(file_path)

# 2. Invocar el skill "pptx"
markdown_content = invoke_skill("pptx", file_path)

# 3. Retornar el markdown
return markdown_content
```

#### Para XLSX (.xlsx, .xls)

```python
# 1. Verificar/instalar skill
if not ensure_skill_installed("xlsx"):
    # Fallback: usar pandas directamente
    return convert_xlsx_with_pandas(file_path)

# 2. Invocar el skill "xlsx"
markdown_content = invoke_skill("xlsx", file_path)

# 3. Retornar el markdown
return markdown_content
```

### Paso 3: Formatos sin skills de Anthropic

Para formatos que no tienen skill especializado (CSV, TXT, RTF, VTT, SRT, HTML, JSON, XML, imágenes),
usa el script Python existente como fallback:

```bash
python3 .github/Skills/TransformDocsToMarkdown/scripts/convert_to_markdown.py \
  "{file_path}" --quiet
```

---

## 🚀 Modo de invocación desde el agente

### Opción A: Invocación directa del skill (recomendado)

Cuando invoques un skill de Anthropic, simplemente menciónalo en tu respuesta y el sistema
de skills lo ejecutará automáticamente:

```
Para convertir este PDF a Markdown, voy a usar el skill `pdf` de Anthropic.
[Aquí el sistema invoca automáticamente el skill]
```

### Opción B: Invocación programática (si necesitas más control)

```python
# Pseudocódigo para el agente
if file_path.endswith(".pdf"):
    # Verificar instalación
    run_command("npx skills list | grep 'anthropics/skills.*pdf'")
    # Si no está instalado
    run_command("npx skills add anthropics/skills@pdf")
    # Invocar skill (delegación automática)
    return f"Usando el skill 'pdf' para procesar {file_path}"
```

---

## ✅ Ventajas de esta arquitectura

1. **Instalación automática:** Los miembros del equipo no necesitan configurar nada manualmente
2. **Maintenance simplificado:** Anthropic mantiene los skills actualizados
3. **Comunidad:** Skills probados por miles de usuarios (50K+ installs cada uno)
4. **Escalabilidad:** Fácil agregar nuevos formatos cuando Anthropic publique skills
5. **Fallbacks robustos:** Si un skill falla, usamos herramientas directas (markitdown, pandoc, pandas)

---

## 📝 Notas de implementación

### Timeouts

```python
# Al instalar skills, usa timeout de 60 segundos
subprocess.run(..., timeout=60)
```

### Manejo de errores

```python
try:
    markdown = invoke_skill("pdf", file_path)
except SkillNotAvailableError:
    # Fallback a herramienta directa
    markdown = convert_pdf_with_markitdown(file_path)
except Exception as e:
    # Loggear error y continuar
    print(f"Error procesando {file_path}: {e}")
    return None
```

### Cache de verificación

Para evitar verificar skills en cada archivo, cachea la información:

```python
_installed_skills = set()

def is_skill_installed(skill_name: str) -> bool:
    if skill_name in _installed_skills:
        return True
    
    skill_path = Path.home() / ".agents" / "skills" / skill_name / "SKILL.md"
    if skill_path.exists():
        _installed_skills.add(skill_name)
        return True
    
    return False
```

---

## 🔄 Migración desde v2.0

Si anteriormente usabas el script `convert_to_markdown.py`:

1. **Mantén el script como fallback** para formatos sin skills de Anthropic
2. **Actualiza el agente** para que priorice los skills de Anthropic
3. **No elimines las dependencias Python** hasta validar que todo funcione correctamente

### Roadmap de deprecación

- **Fase 1** (actual): Skills de Anthropic como método primario, script como fallback
- **Fase 2** (futuro): Solo skills + herramientas directas (markitdown, pandoc, pandas)
- **Fase 3** (más adelante): Deprecar completamente el script Python

---
## ⚠️ Limitaciones conocidas

- **Imágenes dentro de documentos:** Los skills de Anthropic las omiten o las manejan según su implementación interna
- **Documentos con solo imágenes** (`.png`, `.jpg`, etc.): Se puede usar OCR como fallback si el skill no los soporta
- **Archivos protegidos con contraseña:** No soportados por los skills; retornarán error
- **`.doc` y `.ppt` legacy:** Los skills de DOCX y PPTX pueden tener soporte limitado; evalúa convertir a formatos modernos primero

---

## 📚 Referencias

- Skills de Anthropic: [https://skills.sh/anthropics/skills](https://skills.sh/anthropics/skills)
- Skills CLI: [https://skills.sh/](https://skills.sh/)
- Guía de instalación de skills: `npx skills --help`

---