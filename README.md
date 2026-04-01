<div align="center">
  <img src="src/assets/logo-cover.png" width=256></img>
  <h1>ÁgorAI</h1>
  <p><strong>El ágora de las IAs — Orquesta, itera y refina respuestas entre múltiples modelos</strong></p>

[English](README_EN.md) | Español

</div>

## Qué es ÁgorAI

ÁgorAI nace como fork de [ChatALL](https://github.com/ai-shifu/ChatALL), pero con una visión radicalmente diferente: no se trata solo de lanzar un prompt a varios modelos y comparar respuestas, sino de **orquestar conversaciones iterativas entre múltiples IAs**.

Como en el ágora griega — el espacio público donde todos debatían —, aquí los modelos dialogan, se complementan y refinan sus respuestas bajo tu dirección como orquestador.

## El problema

ChatALL permite enviar un prompt a varios bots a la vez, pero ahí termina la historia: recibes respuestas en paralelo y tú decides cuál es mejor. No hay iteración, no hay diálogo entre modelos, no hay refinamiento.

## La solución: orquestación iterativa

ÁgorAI introduce un flujo de trabajo nuevo:

1. **Ronda inicial** — Lanzas tu prompt a los modelos que elijas (ChatGPT, Claude, Gemini, Grok, etc.)
2. **Evaluación** — Revisas las respuestas y seleccionas las más interesantes
3. **Re-inyección** — Alimentas las respuestas seleccionadas a otros modelos (o a los mismos) como contexto para una nueva iteración
4. **Refinamiento** — Repites el ciclo tantas veces como necesites, construyendo respuestas cada vez más elaboradas
5. **Resultado final** — Obtienes una respuesta pulida que ha pasado por múltiples perspectivas de IA

```
         ┌─────────┐
         │ Tu prompt│
         └────┬────┘
              │
    ┌─────────┼─────────┐
    ▼         ▼         ▼
 ┌──────┐ ┌──────┐ ┌──────┐
 │GPT-4 │ │Claude│ │Gemini│   ← Ronda 1
 └──┬───┘ └──┬───┘ └──┬───┘
    │         │        │
    └─────────┼────────┘
              ▼
     ┌────────────────┐
     │  Tú orquestas  │
     │  seleccionas    │
     │  re-inyectas    │
     └───────┬────────┘
              │
    ┌─────────┼─────────┐
    ▼         ▼         ▼
 ┌──────┐ ┌──────┐ ┌──────┐
 │Grok  │ │Claude│ │GPT-4 │   ← Ronda 2..N
 └──────┘ └──────┘ └──────┘
              │
              ▼
      ┌──────────────┐
      │Resultado final│
      └──────────────┘
```

## Casos de uso

- **Investigación profunda** — Contrastar análisis de múltiples modelos y refinar iterativamente
- **Generación de contenido** — Un modelo genera, otro critica, otro reescribe
- **Depuración de código** — Varios modelos proponen soluciones, se combinan las mejores ideas
- **Brainstorming** — Rondas sucesivas de ideas que se alimentan entre sí
- **Fact-checking cruzado** — Verificar afirmaciones pasando respuestas entre modelos

## Estado del proyecto

> 🚧 **En desarrollo activo** — ÁgorAI está en fase inicial de diseño y desarrollo.

### Roadmap

- [ ] Análisis de la base de código de ChatALL
- [ ] Diseño de la interfaz de orquestación
- [ ] Sistema de selección y re-inyección de respuestas
- [ ] Flujo de iteraciones múltiples
- [ ] Historial de cadenas de orquestación
- [ ] Plantillas de flujos predefinidos (debate, refinamiento, crítica)
- [ ] Exportación de sesiones completas

## Stack técnico

- **Base**: [ChatALL](https://github.com/ai-shifu/ChatALL) (Electron + Vue.js)
- **Bots soportados**: Todos los de ChatALL (30+), incluyendo ChatGPT, Claude, Gemini, Grok, Perplexity, Copilot, Groq, y más
- **Plataformas**: Windows, macOS, Linux

## Desarrollo

```bash
npm install
npm run electron:serve
```

Build:

```bash
npm run electron:build
```

## Créditos

- Basado en [ChatALL](https://github.com/ai-shifu/ChatALL) por [ai-shifu](https://github.com/ai-shifu)
- Concepto y desarrollo: [Eduardo Arias Bravo](https://github.com/eduardoddddddd)

## Licencia

Este proyecto hereda la licencia del proyecto original ChatALL.
