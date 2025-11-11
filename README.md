# TestWF

Repositorio de prueba para validar un flujo mínimo de automatización con un agente de OpenAI y revisión humana antes del merge.

## Objetivo del MVP
- Resumir y evaluar los cambios propuestos en cada Pull Request mediante un agente.
- Ejecutar validaciones básicas automáticas (enlaces Markdown, escaneo de secretos, límites de tamaño).
- Publicar comentarios del agente en la Pull Request y dejar la aprobación final en manos de una persona.

## Acciones principales
1. Detectar archivos modificados y el tipo de cambio realizado.
2. Generar un resumen técnico del Pull Request.
3. Ofrecer sugerencias accionables sobre estilo y pruebas mínimas.
4. Ejecutar validaciones automáticas: enlaces Markdown, escaneo de secretos y límites de tamaño por archivo.
5. Publicar los resultados como comentario en la Pull Request y generar el artefacto `.agent/agent_report.json`.

## Instrucciones para el desarrollo
### Preparación del entorno
- Habilita GitHub Actions en el repositorio y registra los secretos `OPENAI_API_KEY` y `GITHUB_TOKEN` con permisos `contents:write` y `pull_requests:write`.
- Asegúrate de que la rama principal sea `main` y crea ramas de trabajo con el patrón `feature/*`, `fix/*` o `chore/*`.

### Estructura mínima esperada
```
.
├─ .github/
│  └─ workflows/
│     └─ agent-ci.yml
├─ .agent/
│  ├─ config.yaml
│  └─ run_agent.py
├─ src/
│  └─ placeholder.txt
├─ docs/
│  └─ decisions.md
├─ CONTRIBUTING.md
├─ CODE_OF_CONDUCT.md
└─ README.md
```

### Configuración del agente
- Define en `.agent/config.yaml` los parámetros del modelo (por defecto `gpt-4.1-mini` con `temperature: 0.2`).
- Ajusta los límites de tamaño (`checks.file_limits.max_single_file_kb`) y los checks disponibles según las necesidades del proyecto.
- El script `run_agent.py` debe:
  - Leer los cambios desde el contexto del repositorio (rama base contra rama actual).
  - Ejecutar los checks configurados.
  - Preparar un prompt con la información relevante y obtener un resumen desde el modelo especificado.
  - Guardar el reporte en `.agent/agent_report.json` y publicar un comentario en la Pull Request.

### Flujo de contribución recomendado
1. Crear una rama `feature/<descriptor>` o equivalente.
2. Realizar commits pequeños con convención `type(scope): mensaje corto` (por ejemplo, `docs(readme): guía`).
3. Abrir la Pull Request contra `main`.
4. Permitir que el workflow `agent-ci.yml` ejecute el agente y suba el comentario automático.
5. Completar la checklist humana: reporte claro, sin secretos, estándares de estilo, riesgos comprendidos y pasos de prueba reproducibles.
6. Un revisor humano aprueba y realiza el merge.

### Buenas prácticas adicionales
- Documenta decisiones arquitectónicas en `docs/decisions.md` utilizando ADRs breves.
- Mantén un registro actualizado de riesgos conocidos y mitigaciones en el reporte del agente.
- Refuerza los checks con linters o pruebas específicas para los lenguajes que se integren en `src/`.

## Roadmap y propuestas de desarrollo
- **Propuesta:** Incorporar una matriz de pruebas parametrizable por lenguaje en `agent-ci.yml` para ampliar la cobertura automática sin sacrificar la simplicidad del MVP.
- Automatizar el etiquetado de Pull Requests según el tipo de cambio detectado.
- Añadir linters específicos por lenguaje y reportar sus resultados en el comentario del agente.

## Historial de cambios
- 2024-05-14: Reestructuración completa del README para clarificar objetivos, acciones e instrucciones de desarrollo; se agregó propuesta de desarrollo inicial.
