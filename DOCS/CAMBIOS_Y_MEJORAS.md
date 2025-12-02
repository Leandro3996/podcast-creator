# Cambios y Mejoras - Fork de podcast-creator

Este documento describe las modificaciones realizadas al fork de `podcast-creator` para mejorar su funcionamiento con Open Notebook.

**Fork:** https://github.com/Leandro3996/podcast-creator
**Original:** https://github.com/lfnovo/podcast-creator
**Fecha:** 2025-12-02

---

## Resumen de Cambios

| Problema | Solución | Archivos Modificados |
|----------|----------|---------------------|
| Límite de 10 segmentos | Aumentado a 30 | `episodes.py`, `app.py`, `profile_manager.py` |
| JSON envuelto en markdown | Nueva función de limpieza | `core.py`, `__init__.py` |
| Modelo inventaba speakers | Prompts reforzados | `outline.jinja`, `transcript.jinja` |

---

## 1. Aumento del Límite de Segmentos

### Problema
El código original limitaba los podcasts a máximo 10 segmentos, lo cual producía podcasts de ~15-20 minutos máximo.

### Solución
Se aumentó el límite a 30 segmentos, permitiendo podcasts de hasta ~1 hora.

### Archivos Modificados

**`src/podcast_creator/episodes.py`** (línea 37-38):
```python
# Antes
if v < 1 or v > 10:
    raise ValueError("Number of segments must be between 1 and 10")

# Después
if v < 1 or v > 30:
    raise ValueError("Number of segments must be between 1 and 30")
```

**`src/podcast_creator/resources/streamlit_app/app.py`** (línea 775):
```python
# Antes
num_segments = st.slider("Number of Segments:", 1, 10, 4, ...)

# Después
num_segments = st.slider("Number of Segments:", 1, 30, 4, ...)
```

**`src/podcast_creator/resources/streamlit_app/utils/profile_manager.py`** (línea 414-415):
```python
# Antes
if num_segments < 1 or num_segments > 10:
    errors.append("Number of segments must be between 1 and 10")

# Después
if num_segments < 1 or num_segments > 30:
    errors.append("Number of segments must be between 1 and 30")
```

### Duración Estimada por Segmentos

| Segmentos | Duración Aproximada |
|-----------|---------------------|
| 10 | ~18 min |
| 15 | ~27 min |
| 20 | ~36 min |
| 25 | ~45 min |
| 30 | ~54 min |

*Nota: La duración real depende del tamaño de cada segmento (short/medium/long).*

---

## 2. Fix para JSON Envuelto en Markdown

### Problema
Algunos modelos LLM (especialmente Gemini 2.0 Flash) devuelven el JSON envuelto en bloques de código markdown:

```
```json
{"transcript": [...]}
```
```

Esto causaba errores de parsing:
```
Invalid json output: ```json
```

### Solución
Se agregó una nueva función `clean_markdown_code_blocks()` que extrae el JSON de los bloques markdown antes del parsing.

### Archivos Modificados

**`src/podcast_creator/core.py`**:

Nueva regex (línea 14):
```python
MARKDOWN_CODE_BLOCK_PATTERN = re.compile(r"^```(?:json)?\s*\n?(.*?)\n?```$", re.DOTALL)
```

Nueva función (líneas 63-95):
```python
def clean_markdown_code_blocks(content: str) -> str:
    """
    Remove markdown code block wrappers from content.

    Some AI models wrap JSON output in markdown code blocks like:
    ```json
    {"key": "value"}
    ```

    This function extracts the content from within those blocks.
    """
    if not isinstance(content, str):
        return str(content) if content is not None else ""

    content = content.strip()

    match = MARKDOWN_CODE_BLOCK_PATTERN.match(content)
    if match:
        return match.group(1).strip()

    return content
```

Integración en `clean_thinking_content()` (línea 122):
```python
def clean_thinking_content(content: str) -> str:
    _, cleaned_content = parse_thinking_content(content)
    # También limpiar bloques markdown que algunos modelos agregan
    cleaned_content = clean_markdown_code_blocks(cleaned_content)
    return cleaned_content
```

**`src/podcast_creator/__init__.py`**:
- Agregado export de `clean_markdown_code_blocks`

---

## 3. Validación Estricta de Speakers

### Problema
Cuando el briefing mencionaba frases como "entrevista a un experto en tecnología", el modelo creaba un speaker llamado "Experto" en lugar de usar los speakers configurados (ej: Carlos y María).

Error típico:
```
Invalid speaker name 'Experto'. Must be one of: Carlos Rodríguez, María García
```

### Solución
Se reforzaron los prompts de outline y transcript con restricciones más explícitas.

### Archivos Modificados

**`src/podcast_creator/resources/prompts/podcast/outline.jinja`**:

Cambio en la sección de speakers:
```jinja
{# Antes #}
The podcast will feature the following speakers:

{# Después #}
The podcast will feature ONLY the following speakers (do NOT invent or add any other speakers):
```

Nuevo bloque de restricción:
```jinja
CRITICAL: You must ONLY use these {{ speakers|length }} speakers: {% for speaker in speakers %}{{ speaker.name }}{% if not loop.last %}, {% endif %}{% endfor %}.
Do NOT create, invent, or reference any other speakers, guests, experts, or characters.
If the briefing mentions interviewing an "expert" or "guest", one of the existing speakers should play that role based on their backstory.
```

Cambio en tips adicionales:
```jinja
{# Antes #}
- If the briefing mentions a guest, include segments for introducing the guest and featuring their expertise.

{# Después #}
- If the briefing mentions a guest or expert, DO NOT add new speakers. Instead, have one of the existing speakers (based on their backstory) take on that expert role.
```

**`src/podcast_creator/resources/prompts/podcast/transcript.jinja`**:

Cambio en la sección de speakers:
```jinja
{# Antes #}
The podcast features the following speakers:

{# Después #}
The podcast features ONLY the following {{ speakers|length }} speakers (NO OTHER SPEAKERS ALLOWED):
```

Nuevo bloque de restricción:
```jinja
⚠️ CRITICAL CONSTRAINT: You may ONLY use these exact speaker names: {{ speaker_names|join(', ') }}.
Do NOT use any other names like "Expert", "Guest", "Host", "Interviewer", etc.
If the content mentions experts or guests, one of the above speakers must play that role.
```

Cambio en guidelines:
```jinja
{# Antes #}
- IMPORTANT: Only use the provided speaker names: {{ speaker_names|join(', ') }}

{# Después #}
- ⚠️ CRITICAL: ONLY use these exact speaker names: {{ speaker_names|join(', ') }}. Using ANY other name will cause an error.
```

---

## Configuración de Open Notebook

Para usar este fork con Open Notebook, se requieren los siguientes cambios:

### `pyproject.toml`
```toml
# Cambiar de:
"podcast-creator>=0.7.0",

# A:
"podcast-creator @ git+https://github.com/Leandro3996/podcast-creator.git@main",
```

### `Dockerfile.single`
Agregar `git` a las dependencias de runtime:
```dockerfile
RUN apt-get update && apt-get upgrade -y && apt-get install -y \
    ffmpeg \
    supervisor \
    curl \
    git \  # <-- Necesario para dependencias de GitHub
    && ...
```

Cambiar `uv sync` para regenerar lock:
```dockerfile
# Antes
RUN uv sync --frozen --no-dev

# Después
RUN uv sync --no-dev
```

### `docker-compose.yml`
Usar build local en lugar de imagen pre-construida:
```yaml
# Antes
open-notebook:
  image: lfnovo/open_notebook:v1-latest-single

# Después
open-notebook:
  build:
    context: ./open-notebook
    dockerfile: Dockerfile.single
```

---

## Commits

1. **`8b1e4f1`** - `feat: increase segment limit and fix JSON markdown parsing`
2. **`4254a7d`** - `fix: enforce strict speaker validation in prompts`

---

## Monitoreo de Logs

Para verificar que los podcasts se generan correctamente:

```bash
# Ver progreso general
docker logs open-notebook --tail 20 -f

# Filtrar solo progreso de podcast
docker logs open-notebook 2>&1 | grep -E "(segment|Batch|clip|Generated|ERROR)"
```

Logs esperados (sin errores):
```
Generating transcript for segment 1/15: ...
Processing batch 1/46 (clips 0-2)
Generating audio clip 0000 for María García
Generating audio clip 0001 for Carlos Rodríguez
Generated audio clip: .../clips/0000.mp3
```

---

## Notas Adicionales

- Los cambios son compatibles con versiones futuras del fork original
- El límite de 30 segmentos fue probado exitosamente hasta 20 segmentos
- La función de limpieza de markdown es transparente (no afecta JSON limpio)
- Los prompts reforzados funcionan con Gemini 2.0 Flash y otros modelos
