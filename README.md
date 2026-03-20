# opencode-dev-workflow

Repositorio de configuración global para [opencode](https://opencode.ai). Contiene agentes reutilizables y protocolos (skills) que el asistente de IA carga automáticamente en cualquier proyecto.

---

## Configuración: apuntar opencode a este directorio

opencode busca su configuración global en `~/.config/opencode/` por defecto. Para usar este repositorio en su lugar, exporta la variable de entorno `OPENCODE_CONFIG_DIR`:

```bash
export OPENCODE_CONFIG_DIR="$HOME/Source/opencode-dev-workflow"
```

Agrega esa línea a tu `~/.zshrc`, `~/.bashrc` o `~/.profile` para que sea permanente:

```bash
echo 'export OPENCODE_CONFIG_DIR="$HOME/Source/opencode-dev-workflow"' >> ~/.zshrc
source ~/.zshrc
```

> **Nota:** ajusta la ruta si clonaste el repo en un lugar diferente.

### Verificar que funciona

Abre opencode en cualquier proyecto. Si los agentes `issue-flow` y `architect` aparecen en el selector de agentes, la configuración es correcta.

---

## Instalación

Este repo usa **bun** para gestionar la dependencia del plugin de opencode.
Los archivos `package.json` y `bun.lock` están en `.gitignore` (son locales, no se versionan).

```bash
git clone https://github.com/<tu-usuario>/opencode-dev-workflow.git ~/Source/opencode-dev-workflow
cd ~/Source/opencode-dev-workflow
bun install
export OPENCODE_CONFIG_DIR="$HOME/Source/opencode-dev-workflow"
```

---

## Utilidades incluidas

### 🔁 issue-flow — Resolución de issues de GitHub de extremo a extremo

**Agente:** `agents/issue-flow.md`  
**Skill:** `skills/resolve-issue/SKILL.md`

Orquesta la resolución completa de un issue de GitHub siguiendo un flujo estructurado en 7 fases, con pausa y aprobación del usuario entre cada una:

| Fase | Acción                                                        |
|------|---------------------------------------------------------------|
| -1   | Detecta sesión existente (resume o reinicia)                  |
| 0    | Carga el issue con `gh issue view`; asigna `@me`              |
| 1    | Triage: clasifica tipo, requisitos, alcance e impacto         |
| 2    | Delega análisis y plan al subagente `architect`               |
| 3    | Crea la rama y delega implementación a `general`              |
| 4    | Ejecuta `make lint` y `make test`                             |
| 5    | Hace el commit y abre el PR con `gh pr create`                |

Al finalizar cada fase, el agente registra en el archivo de historial el modelo usado y los tokens consumidos (input/output) para trazabilidad del coste por paso.

**Cómo usarlo:**

1. Abre opencode dentro del repositorio donde está el issue.
2. Selecciona el agente **issue-flow** en el selector de agentes.
3. Proporciona la URL del issue o su número (`#42`).
4. Aprueba o ajusta el plan en cada pausa.

El agente genera ramas con el formato `issue-<número>-<slug>`, commits con [Conventional Commits](https://www.conventionalcommits.org/) y PRs con una estructura estándar de Summary / Changes / Testing.

---

### 🔍 triage — Análisis estructurado de issues (subagente oculto)

**Agente:** `agents/triage.md`

Subagente oculto invocado por `issue-flow` en la Fase 1. Recibe el contenido del issue y la lista de labels del repositorio como texto, y produce un informe estructurado de triage con:

- Clasificación del tipo de issue y label recomendado
- Requisitos funcionales y no funcionales
- Criterios de aceptación (extraídos o derivados)
- Alcance (en scope / fuera de scope)
- Análisis de impacto (directo, indirecto, datos, API, tests)
- Ambigüedades detectadas

No llama a ninguna herramienta ni modifica nada — trabaja exclusivamente con el contexto pasado en el prompt. El agente orquestador (`issue-flow`) aplica los cambios tras la aprobación del usuario.

---

### 🏛️ architect — Análisis arquitectónico (subagente)

**Agente:** `agents/architect.md`

Subagente de solo lectura invocado por `issue-flow`. Analiza la arquitectura del proyecto, propone decisiones de diseño y genera planes de implementación detallados sin modificar ningún archivo.

No se usa directamente: `issue-flow` lo invoca en la Fase 2.

---

## Estructura del repositorio

```
agents/                        # Definiciones de agentes (.md con front-matter YAML)
│   ├── issue-flow.md          # Agente primario orquestador
│   ├── architect.md           # Subagente de análisis (solo lectura)
│   └── triage.md              # Subagente oculto de triage (sin acceso a herramientas)
skills/                        # Protocolos reutilizables cargados en runtime
│   └── resolve-issue/
│       └── SKILL.md           # Protocolo completo del flujo issue-flow
AGENTS.md                      # Convenciones internas para agentes que operan en este repo
README.md                      # Este archivo
```

---

## Agregar nuevas utilidades

Cada utilidad nueva se compone de un agente + opcionalmente un skill:

```bash
# 1. Crear el agente
touch agents/mi-utilidad.md

# 2. Crear el skill (si tiene protocolo propio)
mkdir -p skills/mi-utilidad
touch skills/mi-utilidad/SKILL.md

# 3. Instalar dependencias (regenera node_modules si hace falta)
bun install
```

Consulta `AGENTS.md` para el formato exacto de front-matter, convenciones de naming y checklist de permisos.

---

## Requisitos

| Herramienta | Uso                                          |
|-------------|----------------------------------------------|
| `bun`       | Instalar dependencias del plugin de opencode |
| `gh`        | CLI de GitHub (usado por `issue-flow`)       |
| `git`       | Control de versiones                         |
| `make`      | Ejecutar lint y tests en proyectos destino   |
