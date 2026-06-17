# CHANGELOG — Skill de Trébol

Historial de cambios al skill de Trébol distribuido vía [skills.sh](https://www.skills.sh). Las fechas corresponden al campo `last_updated` del frontmatter de `SKILL.md`.

Los integradores pueden preguntarle a su asistente con IA *"¿qué cambió en la última versión del skill de Trébol?"* y recibir un resumen consultando este archivo.

## 2026-06-17

- **Pipeline de configuración de plantillas de dictamen**: nuevo sub-pipeline `dictamen-template-pipeline/` integrado dentro del skill de Trébol. Permite configurar plantillas Word (.docx) y PDF de clientes insertando las variables de Trébol en los lugares correctos, para que el endpoint de exportación (`GET`/`POST /v2/verifications/{id}/export/{doc-template-id}`) las autollene. Incluye 6 stages, panel de revisión con 3 personas, helpers Word/PDF y catálogo completo de variables.
- **Endpoint de exportación documentado**: contrato completo del GET y del POST con `key_people` para apoderados; aclarado que la respuesta es `{ download_url }` (no el documento directamente).
- **Catálogo de variables ampliado**: familia `shareholder_person_0_*` (accionista persona física) documentada con campos confirmados contra payload real.

## 2026-05-13

- **Cobertura expandida**: agregados stubs para los otros casos de uso de Trébol — `flows/kyb-colombia.md`, `flows/kyb-eeuu.md`, `flows/hipotecas.md` y `flows/nomina.md`. Cada stub incluye items disponibles, ejemplo de payload, reglas de oro del caso y links a la doc canónica para profundizar. Antes el skill solo tenía walk-through detallado de KYB México; ahora cubre toda la matriz de casos de uso documentados.

## 2026-05-12

- **Migración a skills.sh**: el skill ahora se instala con `npx skills add gotrebol/skills` y se actualiza con `npx skills update`. Soporta múltiples editores con IA (Claude Code, Cursor, Codex, GitHub Copilot, Windsurf y otros). Antes solo funcionaba como plugin de Claude Code.
- **Regla de auto-frescura**: si pasaron más de 30 días desde la última actualización del skill, el asistente le recuerda al dev correr `npx skills update` al final de sus respuestas sobre Trébol.
- **CHANGELOG público**: este archivo. Permite al asistente responder preguntas sobre qué cambió en versiones recientes.

## Versiones anteriores

Para cambios previos a 2026-05-12, ver el historial de commits del repo público:

https://github.com/gotrebol/skills/commits/main/skills/trebol
