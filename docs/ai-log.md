# 📝 Bitácora de IA

Registro de las interacciones con el agente de IA: herramienta y modelos utilizados, prompts clave, errores o alucinaciones y cómo se corrigieron.

## Entrada inicial

- **Fecha:** 22/10/2026
- **Tarea:** Scaffolding del entorno agéntico del proyecto de automatización de n8n (Zapatería Paso Firme).
- **Rama:** `setup-agent-workflow`
- **Agente:** OpenCode
- **Modelo:** Flint 40B Free (proveedor opencode, capa gratuita)
- **Contexto:** Preparar la estructura del repositorio para el desarrollo agéntico de los flujos: `AGENTS.md`, `specs/`, `workflows/`, `scripts/` y esta bitácora.

## Prompts clave

- Creación del `AGENTS.md` adaptado al proyecto: integración n8n ↔ app de la tienda, requisitos técnicos, workflow tipo del proyecto del gimnasio.

## Errores / alucinaciones

- Ninguno en esta fase inicial de scaffolding.

## Decisiones

- El agente no accede a la base de datos de la tienda directamente; todo pasa por la API REST.
- Los importes (`abonado`, `pendiente`, `precio`) los calcula la app y el agente los toma tal cual de las respuestas de la API.
