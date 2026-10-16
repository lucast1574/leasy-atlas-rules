# Skill de Atlas de Lucas

Skill personal de Codex para trabajar exclusivamente en `leasyauto/atlas`: reglas de desarrollo, revisión, CI y capturas de PR, con criterios derivados de reviews de Mikel y Yoel.

## Instalar en Codex

Con acceso a este repositorio privado, pide a Codex:

> Usa skill-installer para instalar la skill de https://github.com/lucast1574/leasy-atlas-rules/tree/main/skills/leasy-atlas-rules

También puedes clonar el repositorio y copiar la carpeta `skills/leasy-atlas-rules` a tu directorio personal de skills de Codex (`~/.codex/skills/` en Linux/macOS o `%USERPROFILE%\.codex\skills\` en Windows, salvo configuración personalizada).

Reinicia Codex después de instalarla. Si ya existe una versión, respáldala y reemplaza la carpeta completa para mantener las referencias sincronizadas.

## Usar

Invoca `$leasy-atlas-rules` en una tarea de Atlas. La skill también puede seleccionarse automáticamente cuando corresponda.

## Contenido

- `SKILL.md`: alcance, reglas esenciales y selección de referencias.
- `references/atlas-rules.md`: reglas de Lucas y contexto del entorno original.
- `references/review-and-ci.md`: implementación, reviews y validación de CI.
- `references/guarantor-review.md`: locks, documentos, países, readiness y modales.
- `references/pr-screenshots.md`: capturas y assets de los PRs.

Las referencias están dentro de `skills/leasy-atlas-rules/`. Los ejemplos de entornos históricos deben comprobarse contra el proyecto actual. Las contraseñas de ejemplo del documento original se sustituyeron por entrada interactiva en esta copia distribuible.

## Actualizar

Los cambios hechos a una copia instalada no se sincronizan automáticamente con GitHub. Actualiza los archivos de este repositorio y vuelve a instalar o reemplazar la carpeta para distribuir una nueva versión.
