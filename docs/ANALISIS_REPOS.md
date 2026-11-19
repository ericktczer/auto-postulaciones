# Análisis comparativo de los dos repos de referencia

Análisis de solo lectura (no se ejecutó nada). Fecha: 2026-11-19.

## 1. Resumen ejecutivo

| | **Auto_job_applier_linkedIn** | **ai-job-search** |
|---|---|---|
| Autor / licencia | GodsScion (clon, rama `feature/multi-source-scoring`), MIT | Mads Lorentzen (`ai-job-search-dev/ai-job-search`), MIT |
| Versión | 26.01.20.5.08 | 1.7.4 (changelog congelado) |
| Naturaleza | Aplicación Python que **postula** | Framework de prompts para Claude Code que **prepara** |
| Motor | Selenium + `undetected_chromedriver` | Claude Code (comandos/skills/subagentes .md) + CLIs Bun/TS + utilidades Python |
| Interfaz | Panel web Flask (`127.0.0.1:5000`) | Ninguna (terminal de Claude Code) |
| ¿Envía postulaciones? | Sí (LinkedIn Easy Apply) | **Nunca** |
| IA | Opcional: OpenAI-compatible, DeepSeek, Gemini (LangGraph) | Solo Claude vía Claude Code |
| Portales | LinkedIn (completo), Indeed (solo escaneo, experimental) | LinkedIn (endpoints guest), freehire.me, 4 portales daneses |
| Persistencia | CSV + `job_scan_report.xlsx` | `seen_jobs.json`, `job_search_tracker.csv`, carpetas por postulación |

## 2. Auto_job_applier_linkedIn

**Raíz real:** `Auto_job_applier_linkedin-main\Auto_job_applier_linkedIn-main\` (carpeta anidada).

### Qué hace
Bot local que, con tu cuenta, busca en LinkedIn, puntúa las ofertas y postula por Easy Apply a las mejores. La rama actual cambió el flujo original a un pipeline en dos fases:

`search -> scan (scrapea título, empresa, descripción, salario, reputación) -> requirements_gate -> scorer -> top_n_to_apply -> apply`

### Estructura clave
- `runAiBot.py`: login y orquestación; modo continuo (`run_non_stop`) con rotación de filtros.
- `app.py`: panel Flask (`/api/config`, `/api/run`, `/api/stop`, `/api/logs`, `/history`).
- `modules/pipeline.py`: el pipeline.
- `modules/sources/` (`base.py` con interfaz `JobSource`; `linkedin_source.py`; `indeed_source.py`).
- `modules/scoring/` (`requirements_gate.py`, `scorer.py`), `modules/ai/` (`connections.py`, `prompts.py`), `modules/reporting/report_writer.py`.
- `config/*.py` (defaults) + `user_config.json` (override desde el panel).
- `tests/` con 15 archivos y fixtures HTML reales.

### Tecnologías
`langchain 1.3.14`, `langchain-openai`, `langchain-google-genai`, `langgraph`; sin versión fijada: selenium, undetected-chromedriver, flask, flask-cors, openpyxl, pyautogui.

### Páginas que automatiza
- **LinkedIn**: búsqueda con filtros, Easy Apply paso a paso. Con `easy_apply_only=False` solo **captura la URL externa**, no llena el formulario externo.
- **Indeed**: experimental, solo escanea; selectores **sin verificar**; no postula.

### Formularios
Reglas por etiqueta (`label_has`, `work_authorization_answer`, `match_answer_to_option`); si no hay respuesta, LangGraph con IA; si sigue sin respuesta, pausa o salta. **Nunca marca** checkboxes de atestación/consentimiento. No hay manejo de CAPTCHA ni checkpoint.

### Puntaje
Determinista: reputación 0.30, salario 0.25, crecimiento 0.20, dificultad 0.25. Compuerta dura por palabras prohibidas, patrocinio de visa, clearance y años de experiencia.

### Limitaciones y riesgos
- Selectores frágiles ante cambios del DOM de LinkedIn; Indeed aún más.
- `undetected-chromedriver` está abandonado (último 3.5.5); con `auto_manage_driver=False` no hay anti-detección.
- ToS de LinkedIn: riesgo de restricción de cuenta (`docs/legal.md`). El límite diario de Easy Apply ronda 25.
- Seguridad: `CORS(app)` abierto mientras `/api/config` devuelve credenciales y claves; credenciales en claro en `user_config.json`/`config/secrets.py`.
- `import modules.open_chrome` abre Chrome al importar (acoplamiento y pruebas difíciles).
- Los defaults (`personals.py`, `questions.py`) traen datos del autor y respuestas como `legally_authorized="Yes"`; si no se cambian, se envía información falsa.
- Con `safe_mode=True` la sesión no persiste; con `pause_before_submit=True` no funciona desatendido.
- Auto-actualizador `git pull` pensado para otro repo/rama; respuesta fija con la URL del repo del autor a "¿cómo se enteró?".
- Historial del panel con ruta fija, regex de experiencia frágil.
- No hay evidencia de uso real en el repo (los logs son de pruebas).

## 3. ai-job-search

**Raíz real:** `ai-job-search-master\ai-job-search-master\`. No es un git clone.

### Qué hace
Conjunto de comandos de Claude Code: `/setup` (perfil), `/scrape` (descubrir ofertas), `/rank` (puntuar en lote), `/apply` (CV + carta personalizados), más `/interview`, `/outcome`, `/gmail-sync`, `/notion-sync`, `/html-report`, `/upskill`, `/add-portal`, etc.

### Flujo de `/apply`
Obtener la oferta, evaluar ajuste, redactar CV y carta en LaTeX, **auditoría factual** por subagente `application-reviewer` contra el perfil, compilar con lualatex/xelatex, verificar PDF (CV de 2 páginas, carta de 1, capa de texto ATS), registrar en tracker con estado `drafted`.

### Scoring (`04-job-evaluation.md`)
- Gates: elegibilidad (ciudadanía/residencia/clearance) e idioma (FAIL/FLAG).
- Cinco dimensiones: habilidades 30 %, experiencia 25 %, cultura 15 %, alineación de carrera 30 %, ubicación PASS/FAIL/FLAG.
- Bandas: Strong 75+, Good 60–74, Moderate 45–59, Weak 30–44, Poor <30. Marcas de urgencia y antigüedad.

### Portales
LinkedIn (endpoints públicos `jobs-guest`), freehire.me (API JSON), Jobindex, Jobnet, Jobbank, Jobdanmark (Dinamarca). **Sin Indeed, Computrabajo ni OCC**; `/add-portal` genera CLIs nuevos. Todo HTTP simple; sin Selenium/Playwright.

### Seguridad
`posting_guard.py` (guardia de integridad contra prompt injection), trust boundary en los prompts, subagentes de solo lectura, allowlist/deny en `.claude/settings.json`. `SECURITY.md` admite que no es un sandbox y que en Windows nativo no hay sandbox de Bash.

### Limitaciones
- Depende por completo de Claude Code (sin capa gratuita) y de LaTeX y Bun; alto costo en tokens (prompt de `/apply` ≈ 42 KB, scraper ≈ 48 KB).
- No envía nada y no tiene interfaz.
- Cobertura pobre fuera de Dinamarca; el HTML scraping se rompe en silencio al cambiar el marcado.
- El scoring lo hace el LLM: no es reproducible ni calibrado.
- Prompts con parches acumulados; en Windows hay que usar `python`, no `python3`.
- ToS de LinkedIn: el propio skill advierte que el acceso automatizado lo incumple.

## 4. Matriz: qué reutilizar

| Capacidad | Auto_job_applier | ai-job-search | Decisión |
|---|---|---|---|
| Postular de verdad | ✅ LinkedIn | ❌ | Tomar el patrón y extenderlo a Indeed MX, Computrabajo y OCC |
| Motor navegador | ✅ Selenium + UC | ❌ HTTP | Selenium, pero evaluar `nodriver`/Selenium con perfil persistente |
| Abstracción por portal | ✅ `JobSource` | ✅ skills por portal | Usar `JobSource` (search/scan/apply) |
| Interfaz web | ✅ Flask | ❌ | Reutilizar; añadir cola de aprobación y restringir CORS |
| Scoring | Determinista simple | LLM, 5 dimensiones + gates | Híbrido: gates deterministas + puntaje LLM con rúbrica |
| CV/carta por oferta | ❌ | ✅ LaTeX + auditoría factual | Tomar la auditoría; salida más simple (PDF/DOCX) |
| Respuesta a formularios | Reglas + LLM | Plantilla de campos | Reglas + Claude con grounding en el perfil |
| Anti prompt-injection | ❌ | ✅ `posting_guard` | Adoptar |
| Dedupe y estado | CSV/Excel | `seen_jobs.json` | SQLite |
| Seguimiento | CSV | tracker + Gmail/Notion | SQLite + panel |
| Proveedor IA | OpenAI/Gemini | Claude (vía CLI) | Anthropic SDK directo |

## 5. Huecos que ninguno cubre
1. Postulación real en **Indeed MX, Computrabajo y OCC**.
2. Manejo de **CAPTCHA / checkpoint** (avisar y pausar).
3. Aprobación humana desde la interfaz (cola para LinkedIn).
4. Secretos fuera de archivos y panel seguro.
5. Pruebas contra los DOM reales de los portales mexicanos y detección de cambios de selectores.
6. Contexto mexicano: español como idioma principal, salario en MXN, ubicación/CDMX/remoto.

## 6. Riesgos a tener presentes
- Los cuatro portales tienen términos de uso que restringen la automatización; el modo automático puede causar bloqueo de cuenta, CAPTCHAs o postulaciones erróneas.
- Enviar datos falsos o inventados por la IA en una postulación tiene consecuencias reales: el sistema debe basarse solo en el perfil verificado.
- Las ofertas son texto no confiable que puede contener instrucciones dirigidas a la IA.
