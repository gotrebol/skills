# Trébol — Skills oficiales

Este repo contiene los skills oficiales de [Trébol](https://gotrebol.com) distribuidos vía [skills.sh](https://www.skills.sh) para editores con IA (Claude Code, Cursor, Codex, GitHub Copilot, Windsurf y otros).

## Instalar

```bash
npx skills add gotrebol/skills
```

## Skills disponibles

- [`skills/trebol`](./skills/trebol/) — Integración con la API de Trébol (autenticación, endpoints, flujos KYB, hipotecas, nómina, webhooks, widget) y configuración de plantillas de dictamen jurídico de clientes.

## Documentación

- Guía oficial de integración: https://docs.gotrebol.com
- Cómo instalar el skill en tu editor: https://docs.gotrebol.com/guia-devs/skill

## Cómo se mantiene este repo

El desarrollo se hace en el repo privado `gotrebol/docs` (carpeta `skills/trebol/`). Un GitHub Action sincroniza automáticamente el contenido aquí cuando se mergea a `main`. Este repo es público solo para que `npx skills add` funcione — no edites directamente acá, los cambios serán sobrescritos por el sync.

## Soporte

- Reportar bugs: [issues](https://github.com/gotrebol/skills/issues)
- Email: help@gotrebol.com

## Licencia

MIT
