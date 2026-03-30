# opencode-dev-workflow

Repositorio de configuración global para [opencode](https://opencode.ai). Contiene agentes reutilizables y protocolos (skills) que el asistente de IA carga automáticamente en cualquier proyecto.

---

## Instalación

```bash
git clone https://github.com/<tu-usuario>/opencode-dev-workflow.git ~/Source/opencode-dev-workflow
cd ~/Source/opencode-dev-workflow
bun install
```

---

## Activación

opencode busca su configuración global en `~/.config/opencode/` por defecto. Para usar este repositorio en su lugar:

```bash
export OPENCODE_CONFIG_DIR="$HOME/Source/opencode-dev-workflow"
```

Agrégalo permanentemente a tu shell:

```bash
echo 'export OPENCODE_CONFIG_DIR="$HOME/Source/opencode-dev-workflow"' >> ~/.zshrc
source ~/.zshrc
```

Para verificar que funciona, abre opencode en cualquier proyecto — los agentes `issue-flow` y `architect` deben aparecer en el selector.

---

## Cómo usar: issue-flow

1. Abre opencode dentro del repositorio donde está el issue.
2. Selecciona el agente **issue-flow** en el selector de agentes.
3. Proporciona la URL del issue o su número (`#42`).
4. Aprueba o ajusta el plan en cada pausa.

El agente sigue un flujo de 7 fases con pausa y aprobación entre cada una:

| Fase | Acción |
|------|--------|
| -1 | Detecta sesión existente (resume o reinicia) |
| 0 | Carga el issue; pregunta si sincronizar con main; verifica AGENTS.md; carga configuración del proyecto |
| 1 | Triage: clasifica tipo, requisitos, alcance e impacto |
| 2 | Delega análisis y plan al subagente `architect`; publica el plan completo en el issue |
| 3 | Propone nombre de rama según convenciones del proyecto; implementa; compila |
| 4 | Ejecuta lint y tests (detectados desde Makefile o README); corrige errores automáticamente |
| 5 | Propone commit message según convenciones; abre el PR |

---

## Agentes incluidos

| Agente | Modo | Descripción |
|--------|------|-------------|
| `issue-flow` | Primario | Orquestador principal. Dirige el flujo completo de resolución de issues. |
| `architect` | Subagente | Análisis arquitectónico de solo lectura. Produce planes de implementación. |
| `triage` | Subagente oculto | Clasificación y análisis de requisitos. Sin acceso a herramientas. |

---

## Estructura del repositorio

```
agents/
├── issue-flow.md          # Agente primario orquestador
├── architect.md           # Subagente de análisis (solo lectura)
└── triage.md              # Subagente oculto de triage (sin herramientas)
skills/
└── resolve-issue/
    └── SKILL.md           # Protocolo completo del flujo issue-flow
AGENTS.md                  # Convenciones para agentes que operan en este repo
README.md                  # Este archivo
```

---

## Añadir nuevas utilidades

Cada utilidad nueva se compone de un agente + opcionalmente un skill:

```bash
# 1. Crear el agente
touch agents/mi-utilidad.md

# 2. Crear el skill (si tiene protocolo propio)
mkdir -p skills/mi-utilidad
touch skills/mi-utilidad/SKILL.md

# 3. Instalar dependencias
bun install
```

Consulta `AGENTS.md` para el formato de front-matter, convenciones de naming y checklist de permisos.

---

## Requisitos

| Herramienta | Uso |
|-------------|-----|
| `bun` | Instalar dependencias del plugin de opencode |
| `gh` | CLI de GitHub (usado por `issue-flow`) |
| `git` | Control de versiones |
| `make` | Ejecutar lint y tests en proyectos destino (si aplica) |
