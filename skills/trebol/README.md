# Trébol — Skill para editores con IA

Skill oficial de [Trébol](https://gotrebol.com) para editores con IA (Claude Code, Cursor, Codex, GitHub Copilot, Windsurf y otros soportados por [skills.sh](https://www.skills.sh)). Trébol automatiza procesos de back office sin importar la complejidad. Este skill le da a tu editor con IA conocimiento embebido de la API de Trébol — autenticación, endpoints, flujos, widget, webhooks y errores comunes — para que la integración sea más rápida.

## Instalar

En tu terminal, ejecuta:

```bash
npx skills add gotrebol/skills
```

skills.sh detecta automáticamente tu editor con IA y configura el skill. Después prueba:

> ¿Cómo creo una verificación KYB para una empresa mexicana?

## Qué incluye

- **Autenticación**: cómo usar `x-api-key`, gestión y rotación de keys
- **Flujos por caso de uso**: KYB México (detallado), KYB Colombia, KYB EEUU, hipotecas y nómina (todos en Beta los últimos tres), widget, webhooks
- **Referencia**: endpoints más usados y errores comunes
- **OpenAPI**: especificación completa para consultas detalladas (cubre todos los endpoints/casos, incluso si no hay walk-through curado)

## Actualizar

```bash
npx skills update
```

Este comando actualiza todos los skills instalados. El skill incluye una regla de auto-frescura: si pasaron más de 30 días desde la última actualización, tu asistente te recordará correr este comando al final de sus respuestas sobre Trébol.

## Desinstalar

```bash
npx skills remove trebol
```

## Soporte

- [Documentación completa](https://docs.gotrebol.com)
- [Reportar bugs](https://github.com/gotrebol/skills/issues)
- [help@gotrebol.com](mailto:help@gotrebol.com)

## Notas para mantenedores

> **Dónde se desarrolla.** Este skill se desarrolla en el repo privado `gotrebol/docs` (carpeta `skills/trebol/`). Un GitHub Action sincroniza automáticamente el contenido al repo público `gotrebol/skills` (estructura: `gotrebol/skills/skills/trebol/`) cuando se mergea a `main`. Los clientes externos instalan desde el repo público — nunca tocan `gotrebol/docs`.

El skill empaqueta versiones curadas de varios documentos canónicos del repo `gotrebol/docs`. Antes de actualizar `last_updated` en `SKILL.md`, sincronizar:

| Archivo del skill | Fuente canónica en `docs/` |
|---|---|
| `reference/openapi.yaml` | `api-reference/openapi.yaml` |
| `flows/webhooks.md` | `guia-devs/webhooks.mdx` (versión curada) |
| `flows/kyb-mexico.md` | `guia-devs/uso-kyb/mexico/*.mdx` (versión curada) |
| `flows/kyb-colombia.md` | `guia-devs/uso-kyb/colombia/*.mdx` (stub: links + tips) |
| `flows/kyb-eeuu.md` | `guia-devs/uso-kyb/eeuu/*.mdx` (stub: links + tips) |
| `flows/hipotecas.md` | `guia-devs/uso-hipotecas/*.mdx` (stub: links + tips) |
| `flows/nomina.md` | `guia-devs/uso-nomina/*.mdx` (stub: links + tips) |
| `flows/widget.md` | `guia-devs/crear-verificaciones/via-widget/*.mdx` (versión curada) |
| `auth.md` | `guia-devs/conectarse.mdx` y `guia-devs/gestion-api-keys.mdx` |
| `reference/endpoints.md` | Resumen de top endpoints del openapi |
| `reference/errors.md` | `guia-devs/errores.mdx` y troubleshooting derivado |

Sincronización mínima del OpenAPI:

```bash
cp api-reference/openapi.yaml skills/trebol/reference/openapi.yaml
```

Para los `.md`: revisar si los flujos canónicos cambiaron (HMAC, eventos, items, country handling) y propagar los cambios al skill antes de actualizar `last_updated`.

### Publicar una nueva versión

1. Sincronizar archivos según tabla anterior
2. Actualizar `last_updated: YYYY-MM-DD` en el frontmatter de `SKILL.md` con la fecha del release (solo si los cambios son significativos para integradores — no para typos o reordenamientos)
3. Agregar una entrada al `CHANGELOG.md` con la misma fecha y un resumen corto del cambio (qué cambió y por qué le importa al integrador). Esto permite que el asistente con IA responda preguntas tipo "¿qué cambió en la última versión?" con información útil.
4. Commit y merge a `main` del repo `gotrebol/docs`
5. El GitHub Action `sync-skill-to-public.yml` sincroniza automáticamente el contenido al repo público `gotrebol/skills` (bajo `skills/trebol/`). Verificar en la pestaña Actions que el sync corrió sin errores.
6. Los clientes con el skill instalado ven los cambios cuando corran `npx skills update`. Si el `last_updated` quedó >30 días atrás, el propio skill les recordará correr ese comando

## Licencia

MIT
