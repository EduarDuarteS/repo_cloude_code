# El "God Mode" Setup: Claude Code CLI + Google Gemini 🚀

Esta es la guía definitiva para orquestar `Claude Code CLI` utilizando los LLMs de Google Gemini a través de LiteLLM, habiendo solucionado todas las limitaciones de esquema (Schema Mismatch) entre Anthropic y Google.

## Los 3 Bloqueadores Técnicos Resueltos
1. **Invalid tool parameters:** Gemini a menudo corrompía la validación Zod de Anthropic generando JSONs multi-línea o enviando parámetros booleanos ocultos.
2. **Error writing file:** El sistema nativo de Claude protegía excesivamente la escritura, y los modelos carecían de adaptación para el dialecto local de Windows.
3. **Ghost 404 Routing:** Una colisión del puerto `v1` duplicado generado por la memoria de sesión del CLI de Claude.

---

## 🛠️ Implementación en 3 Minutos

### 1. El Gateway Proxy (`config.yaml`)
Crea este archivo en la raíz. **Manejo SRE: No uses NINGUN acento (`á`, `é`, `ñ`) en los comentarios o LiteLLM (CP1252) crasheará en Windows con un UnicodeDecodeError.**

```yaml
model_list:
  # Capture any Claude Code attempt
  - model_name: gemini-3.1-pro
    litellm_params:
      # THE KEY: Model optimized for external tool schemas
      model: gemini/gemini-3.1-pro-preview-customtools
      api_key: "os.environ/GEMINI_API_KEY"
      # Hallucination control
      drop_params: true
      # Force Gemini to "think" before generating JSON
      reasoning_effort: high

litellm_settings:
  drop_params: true
  set_verbose: false
```

### 2. Inyección de Contexto Dinámico (`CLAUDE.md`)
Crea este archivo exacto en la raíz del proyecto. **Manejo SRE: Esto actúa como un parche temporal de memoria. Le enseña a la IA a burlar las herramientas rotas usando fallbacks del sistema operativo.**

```markdown
# Reglas de Interacción de Herramientas
- Cuando uses herramientas de edición o creación de archivos, el parámetro que indica la ruta del archivo SIEMPRE debe llamarse `path`, NUNCA `filename` o `file_path`.
- Ejecuta los comandos asumiendo un entorno de PowerShell en Windows. Usa comandos compatibles como `Get-Content` en lugar de `cat`.
```
*(Nota: Gracias a esto, si Gemini recibe un "Error writing file", usará `Bash` automáticamente con `Set-Content` para inyectar archivos sin romper el proxy).*

### 3. Entorno de Ejecución (Terminales)

Abre dos terminales de PowerShell en tu proyecto:

**Terminal 1 (Levanta LiteLLM)**
```powershell
$env:GEMINI_API_KEY="TU_API_KEY_REAL_AQUI"
litellm --config config.yaml
```

**Terminal 2 (Levanta la CLI de Claude)**
Configura el dialecto puro de la API. **Manejo SRE: ANTHROPIC_BASE_URL no debe tener `/v1` al final, de lo contrario la CLI ruteará hacia `/v1/v1/messages` (404 Not Found).**

```powershell
$env:CLAUDE_CODE_USE_VERTEX="0"
# ROOT HOST SOLAMENTE:
$env:ANTHROPIC_BASE_URL="http://localhost:4000"
$env:ANTHROPIC_AUTH_TOKEN="sk-ant-dev-gemini-2026"

# Forzar ejecuccion sin prompts manuales (Automatizacion Total):
$env:CLAUDE_CODE_PERMISSION_MODE="bypassPermissions"

# Apagamos las herramientas Beta complejas de Anthropic (Computer Use, Bash avanzado) 
# que Gemini no parsea bien:
$env:CLAUDE_CODE_DISABLE_EXPERIMENTAL_BETAS="1"
# GOD MODE: Saltar confirmaciones de escritura/lectura y comandos
$env:CLAUDE_CODE_PERMISSION_MODE="bypassPermissions"

# Lanzamos la CLI asegurándonos del Flag Correcto (--model, NO -m)
claude --model gemini-3.1-pro --permission-mode bypassPermissions
```

*(Tip de supervivencia: Si cambias alguna variable y Claude sigue arrojando errores o URLs viejas, escribe `/clear` en el prompt para vaciar su persistencia local).*
