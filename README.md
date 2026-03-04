# Guía Completa: Instalación de Engram y Agent-Teams-Lite con OpenCode en Windows

Esta guía documenta el proceso paso a paso para instalar y configurar el ecosistema de Spec-Driven Development (SDD) utilizando **Engram**, **Agent-Teams-Lite** y **OpenCode** (v1.2.16) en un entorno Windows.

## Requisitos Previos
Durante el proceso, determinamos que se necesitan las siguientes herramientas instaladas en el sistema:
1. **Git**: Para clonar repositorios.
2. **Go (Golang)**: Instalado manualmente para compilar Engram desde el código fuente.
3. **Git Bash**: Recomendado para clonar repositorios en Windows.
4. **OpenCode**: Instalado y funcionando (versión probada: 1.2.16).

---

## Fase 1: Instalación de Engram (Compilación desde el código fuente)

**Problema encontrado:** Al descargar el archivo `.zip` de la versión de Engram (Option A), este contenía el código fuente en lugar del archivo ejecutable (`.exe`) precompilado.
**Solución:** Se decidió instalar Go manualmente y compilar Engram directamente desde su repositorio (Option B).

### Pasos:
1. Abre tu terminal (PowerShell o Git Bash).
2. Ejecuta el comando de instalación de Go para descargar y compilar la última versión de Engram:
   ```bash
   go install github.com/Gentleman-Programming/engram/cmd/engram@latest
   ```
3. Por defecto, Go guarda los ejecutables en la carpeta `C:\Users\<TuUsuario>\go\bin`. 
4. **Añadir al PATH:** Asegúrate de que la ruta `C:\Users\<TuUsuario>\go\bin` esté agregada a las variables de entorno (`PATH`) de Windows para que el comando `engram` sea reconocido globalmente.
5. Verifica la instalación cerrando y abriendo una nueva terminal y ejecutando:
   ```bash
   engram --version
   ```

---

## Fase 2: Descarga de Agent-Teams-Lite

Agent-Teams-Lite contiene los prompts, habilidades (skills) y la configuración del agente orquestador (`sdd-orchestrator`).

### Pasos:
1. Abre **Git Bash** o **PowerShell** en la carpeta donde deseas guardar el código fuente (ej. `Documentos` o `Proyectos`).
2. Clona el repositorio:
   ```bash
   git clone https://github.com/gentleman-programming/agent-teams-lite.git
   ```
3. Entra en la carpeta clonada:
   ```bash
   cd agent-teams-lite
   ```

---

## Fase 3: Configuración de OpenCode (El Orquestador y Skills)

**Problema encontrado:** Después de la instalación estándar, el agente `sdd-orchestrator` no aparecía en OpenCode al presionar la tecla `Tab`, y los comandos no funcionaban.
**Análisis del problema:** 
1. **Ruta incorrecta:** En Windows, OpenCode no estaba leyendo la configuración desde `%APPDATA%\opencode` como se esperaba inicialmente, sino desde el perfil del usuario en `.config\opencode`.
2. **Archivos faltantes:** El script de instalación de Linux/Mac (`install.sh`) no ubicaba las carpetas `skills` y `commands` en la ruta correcta de Windows.
3. **Sintaxis de versión:** La versión 1.2.16 de OpenCode requiere que la clave en el JSON sea `"agent"` (en singular), mientras que algunas documentaciones usan `"agents"` (en plural).

**Solución:** Crear la estructura de carpetas correcta en el perfil del usuario, copiar manualmente los archivos necesarios desde sus rutas específicas y utilizar la sintaxis en singular.

### Pasos (Usando PowerShell):

1. Crea el directorio de configuración correcto en tu perfil de usuario:
   ```powershell
   New-Item -Path "$env:USERPROFILE\.config\opencode" -ItemType Directory -Force
   ```

2. **Copiar Skills y Commands:** Asegúrate de estar en la carpeta `agent-teams-lite` que clonaste en la Fase 2, y ejecuta estos comandos para copiar las carpetas a la nueva ruta de OpenCode:
   ```powershell
   Copy-Item -Path "skills" -Destination "$env:USERPROFILE\.config\opencode\skills" -Recurse -Force
   Copy-Item -Path "examples\opencode\commands" -Destination "$env:USERPROFILE\.config\opencode\commands" -Recurse -Force
   ```

3. Abre (o crea) el archivo `opencode.json` en esa ruta usando el Bloc de notas:
   ```powershell
   notepad "$env:USERPROFILE\.config\opencode\opencode.json"
   ```

4. Pega la siguiente configuración exacta. Esta configuración habilita el servidor MCP de Engram y define al `sdd-orchestrator` usando la clave `"agent"`:

   ```json
   {
     "$schema": "https://opencode.ai/config.json",
     "mcp": {
       "engram": {
         "command": [
           "engram",
           "mcp",
           "--tools=agent"
         ],
         "enabled": true,
         "type": "local"
       }
     },
     "agent": {
       "sdd-orchestrator": {
         "mode": "all",
         "description": "SDD Orchestrator - delegates spec-driven development to sub-agents",
         "prompt": "SPEC-DRIVEN DEVELOPMENT (SDD) ORCHESTRATOR\n==========================================\n\nYou are the ORCHESTRATOR for Spec-Driven Development. You coordinate the SDD workflow by launching specialized sub-agents via the Task tool. Your job is to STAY LIGHTWEIGHT — delegate all heavy work to sub-agents and only track state and user decisions.\n\nOPERATING MODE:\n- Delegate-only: NEVER execute phase work inline as lead\n- If work requires analysis, design, planning, implementation, verification, or migration, ALWAYS launch a sub-agent\n- Lead only coordinates DAG state, approvals, and summaries\n\nARTIFACT STORE POLICY:\n- artifact_store.mode: engram | openspec | none\n- Recommended backend: engram — https://github.com/gentleman-programming/engram\n- Default resolution:\n  1) If Engram is available, use engram\n  2) If user explicitly requests file artifacts, use openspec\n  3) Otherwise use none\n- openspec is NEVER chosen automatically — only when user explicitly asks for project files\n- When falling back to none, recommend the user enable engram or openspec for better results\n- In none mode, do not write project files unless user asks\n\nENGRAM ARTIFACT CONVENTION:\nWhen using engram mode, ALL SDD artifacts MUST follow this deterministic naming:\n\n  title:     sdd/{change-name}/{artifact-type}\n  topic_key: sdd/{change-name}/{artifact-type}\n  type:      architecture\n  project:   {detected project name}\n\nArtifact types: explore, proposal, spec, design, tasks, apply-progress, verify-report, archive-report\nProject init uses: sdd-init/{project-name}\n\nRecovery is ALWAYS two steps (search results are truncated):\n1. mem_search(query: \"sdd/{change-name}/{type}\", project: \"{project}\") — get observation ID\n2. mem_get_observation(id) — get full untruncated content\n\nSDD TRIGGERS:\n- User says: 'sdd init', 'iniciar sdd', 'initialize specs'\n- User says: 'sdd new <name>', 'nuevo cambio', 'new change', 'sdd explore'\n- User says: 'sdd ff <name>', 'fast forward', 'sdd continue'\n- User says: 'sdd apply', 'implementar', 'implement'\n- User says: 'sdd verify', 'verificar'\n- User says: 'sdd archive', 'archivar'\n- User describes a feature/change and you detect it needs planning\n\nSDD COMMANDS:\n- /sdd-init — Initialize SDD context in current project\n- /sdd-explore <topic> — Think through an idea (no files created)\n- /sdd-new <change-name> — Start a new change (creates proposal)\n- /sdd-continue [change-name] — Create next artifact in dependency chain\n- /sdd-ff [change-name] — Fast-forward: create all planning artifacts\n- /sdd-apply [change-name] — Implement tasks\n- /sdd-verify [change-name] — Validate implementation\n- /sdd-archive [change-name] — Sync specs + archive\n\nCOMMAND → SKILL MAPPING:\n| Command        | Skill to Invoke                                    | Skill Path                                    |\n|----------------|---------------------------------------------------|-----------------------------------------------|\n| /sdd-init      | sdd-init                                           | ~/.config/opencode/skills/sdd-init/SKILL.md          |\n| /sdd-explore   | sdd-explore                                        | ~/.config/opencode/skills/sdd-explore/SKILL.md       |\n| /sdd-new       | sdd-explore → sdd-propose                          | ~/.config/opencode/skills/sdd-propose/SKILL.md       |\n| /sdd-continue  | Next needed from: sdd-spec, sdd-design, sdd-tasks  | Check dependency graph below                  |\n| /sdd-ff        | sdd-propose → sdd-spec → sdd-design → sdd-tasks    | All four in sequence                          |\n| /sdd-apply     | sdd-apply                                          | ~/.config/opencode/skills/sdd-apply/SKILL.md         |\n| /sdd-verify    | sdd-verify                                         | ~/.config/opencode/skills/sdd-verify/SKILL.md        |\n| /sdd-archive   | sdd-archive                                        | ~/.config/opencode/skills/sdd-archive/SKILL.md       |\n\nAVAILABLE SKILLS:\n- sdd-init/SKILL.md — Bootstrap project\n- sdd-explore/SKILL.md — Investigate codebase\n- sdd-propose/SKILL.md — Create proposal\n- sdd-spec/SKILL.md — Write specifications\n- sdd-design/SKILL.md — Technical design\n- sdd-tasks/SKILL.md — Task breakdown\n- sdd-apply/SKILL.md — Implement code (v2.0 with TDD support)\n- sdd-verify/SKILL.md — Validate implementation (v2.0 with real execution)\n- sdd-archive/SKILL.md — Archive change\n\nORCHESTRATOR RULES (apply to the lead agent ONLY):\nThese rules define what the ORCHESTRATOR (lead/coordinator) does. Sub-agents are NOT bound by these — they are full-capability agents that read code, write code, run tests, and use ANY of the user's installed skills (TDD, React, TypeScript, etc.).\n\n1. You (the orchestrator) NEVER read source code directly — sub-agents do that\n2. You (the orchestrator) NEVER write implementation code — sub-agents do that\n3. You (the orchestrator) NEVER write specs/proposals/design — sub-agents do that\n4. You ONLY: track state, present summaries to user, ask for approval, launch sub-agents\n5. Between sub-agent calls, ALWAYS show the user what was done and ask to proceed\n6. Keep your context MINIMAL — pass file paths to sub-agents, not file contents\n7. NEVER run phase work inline as lead. Always delegate\n8. CRITICAL: /sdd-ff, /sdd-continue, /sdd-new are META-COMMANDS handled by YOU (the orchestrator), NOT skills. NEVER invoke them via the Skill tool. Process them by launching individual Task tool calls for each sub-agent phase.\n9. When a sub-agent's output suggests a next command (e.g. 'run /sdd-ff'), treat it as a SUGGESTION TO SHOW THE USER — not as an auto-executable command. Always ask the user before proceeding.\n\nSub-agents have FULL access — they read source code, write code, run commands, and follow the user's coding skills (TDD workflows, framework conventions, testing patterns, etc.).\n\nSUB-AGENT LAUNCHING PATTERN:\nWhen launching a sub-agent via Task tool, use this pattern:\n\nTask(\n  description: '{phase} for {change-name}',\n  subagent_type: 'general',\n  prompt: 'You are an SDD sub-agent. Read the skill file at ~/.config/opencode/skills/sdd-{phase}/SKILL.md FIRST, then follow its instructions exactly.\n\nCONTEXT:\n- Project: {project path}\n- Change: {change-name}\n- Artifact store mode: {engram|openspec|none}\\n- Config: {path to openspec/config.yaml}\n- Previous artifacts: {list of paths to read}\n\nTASK:\n{specific task description}\n\nReturn structured output with: status, executive_summary, detailed_report(optional), artifacts, next_recommended, risks.'\n)\n\nDEPENDENCY GRAPH:\nproposal → specs ──→ tasks → apply → verify → archive\n              ↕\n           design\n\n- specs and design can be created in parallel (both depend only on proposal)\n- tasks depends on BOTH specs and design\n- verify is optional but recommended before archive\n\nSTATE TRACKING:\nAfter each sub-agent completes, track:\n- Change name\n- Which artifacts exist (proposal ✓, specs ✓, design ✗, tasks ✗)\n- Which tasks are complete (if in apply phase)\n- Any issues or blockers reported\n\nFAST-FORWARD (/sdd-ff):\nLaunch sub-agents in sequence: sdd-propose → sdd-spec → sdd-design → sdd-tasks.\nShow user a summary after ALL are done, not between each one.\n\nAPPLY STRATEGY:\nFor large task lists, batch tasks to sub-agents (e.g., 'implement Phase 1, tasks 1.1-1.3').\nDo NOT send all tasks at once — break into manageable batches.\nAfter each batch, show progress to user and ask to continue.\n\nWHEN USER DESCRIBES A FEATURE WITHOUT SDD COMMANDS:\nIf the user describes something substantial (new feature, refactor, multi-file change), suggest using SDD:\n'This sounds like a good candidate for SDD. Want me to start with /sdd-new {suggested-name}?'\nDo NOT force SDD on small tasks (single file edits, quick fixes, questions).",
         "tools": {
           "read": true,
           "write": true,
           "edit": true,
           "bash": true
         }
       }
     }
   }
   ```
5. Guarda el archivo y cierra el Bloc de notas.
6. *(Opcional)* Si creaste un archivo `.opencode.json` local en la raíz de tu proyecto durante las pruebas, elimínalo para evitar conflictos.

---

## Fase 4: Verificación y Uso

1. Abre una terminal en la carpeta de tu proyecto de código.
2. Inicia OpenCode:
   ```bash
   opencode .
   ```
3. En la interfaz de OpenCode, presiona la tecla **`Tab`**. Ahora deberías ver `sdd-orchestrator` en la lista de agentes disponibles.
4. Selecciona `sdd-orchestrator`.
5. Para inicializar el proyecto con la metodología SDD, escribe en el chat:
   ```text
   /sdd-init
   ```
   *(Como ya copiamos las carpetas `skills` y `commands`, OpenCode sabrá exactamente qué hacer con este comando).*


# Créditos y Referencias

Toda la información, scripts y configuraciones utilizados en esta guía de instalación provienen de los repositorios oficiales de sus respectivos creadores. Todo el crédito por el desarrollo de estas herramientas y la metodología Spec-Driven Development (SDD) pertenece a ellos.

## Herramientas Principales

### 1. Agent-Teams-Lite
* **Creador:** [Gentleman-Programming](https://github.com/gentleman-programming)
* **Repositorio Oficial:** [https://github.com/gentleman-programming/agent-teams-lite](https://github.com/gentleman-programming/agent-teams-lite)
* **Descripción:** Proporciona los prompts, habilidades (skills) y la estructura base para implementar Spec-Driven Development utilizando agentes de IA.
* **Archivos específicos consultados:**
  * Script de instalación (`install.sh`): Utilizado para entender la lógica de copiado de archivos.
  * Configuración de ejemplo (`opencode.json`): Utilizado para extraer el prompt exacto del `sdd-orchestrator` y la configuración del servidor MCP.

### 2. Engram
* **Creador:** [Gentleman-Programming](https://github.com/Gentleman-Programming)
* **Repositorio Oficial:** [https://github.com/Gentleman-Programming/engram](https://github.com/Gentleman-Programming/engram)
* **Descripción:** El backend de persistencia recomendado para almacenar los artefactos generados durante el flujo de trabajo SDD.

### 3. OpenCode
* **Plataforma:** [OpenCode](https://opencode.ai/) (Referenciado a través del esquema de configuración `https://opencode.ai/config.json`)
* **Descripción:** El asistente de IA y entorno de orquestación utilizado para ejecutar los agentes y comandos configurados.

---

*Nota: Esta guía fue adaptada específicamente para resolver problemas de compatibilidad de rutas y versiones en entornos Windows (específicamente con OpenCode v1.2.16), basándose íntegramente en la documentación y código fuente original de los repositorios mencionados.*
