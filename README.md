# Análisis Arquitectónico de ChatALL → ÁgorAI

**Fecha**: 1 de abril de 2026  
**Repo**: https://github.com/eduardoddddddd/AgorAI  
**Objetivo**: Entender la base de código para transformarla en un orquestador iterativo de IAs con integración MCP

---

## 1. Stack Técnico

| Componente | Tecnología | Versión |
|---|---|---|
| Framework UI | Vue.js | 3.5.16 |
| Estado | Vuex | 4.1.0 |
| Desktop | Electron | 33.4.11 |
| Base de datos local | Dexie (IndexedDB) | 4.0.11 |
| Persistencia Vuex | vuex-persist + localForage | - |
| UI Kit | Vuetify | 3.8.8 |
| LLM Integration | LangChain | 0.3.27 |
| Markdown | v-md-editor | 2.3.18 |
| Build | electron-builder | 25.1.8 |
| i18n | vue-i18n | 11.1.5 |

**Dependencias LangChain por proveedor:**
- `@langchain/openai` (GPT)
- `@langchain/anthropic` (Claude)
- `@langchain/google-genai` (Gemini)
- `@langchain/groq` (Groq)
- `@langchain/cohere` (Cohere)
- `@langchain/xai` (Grok)

---

## 2. Estructura de Directorios

```
src/
├── App.vue                 # Componente raíz — orquesta layout completo
├── background.js           # Proceso principal Electron (ventana, proxy, IPC)
├── main.js                 # Entry point Vue
├── menu.js                 # Menú nativo Electron
├── theme.js                # Sistema de temas (claro/oscuro)
├── update.js               # Auto-actualización
├── bots/                   # ★ CORE — Todos los bots de IA
│   ├── Bot.js              # Clase base abstracta
│   ├── LangChainBot.js     # Clase intermedia para bots API vía LangChain
│   ├── index.js            # Registro global de todos los bots (array `all`)
│   ├── openai/             # GPT-4, GPT-4o, o1, o3, o4-mini, 4.5...
│   ├── anthropic/          # Claude Opus, Sonnet, Haiku, 3.7
│   ├── google/             # Gemini, Gemini Adv, Bard
│   ├── groq/               # Gemma, Llama 3, Llama 4
│   ├── xai/                # Grok 2, 3, 3-mini
│   ├── cohere/             # Command R, Aya 23
│   ├── microsoft/          # Bing Chat, Azure OpenAI
│   ├── huggingface/        # HuggingChat, Gradio
│   ├── moonshot/           # Kimi
│   ├── baidu/              # Wenxin Qianfan (ERNIE)
│   ├── zhipu/              # ChatGLM
│   └── poe/                # Poe
├── components/
│   ├── Messages/           # ★ Renderizado de mensajes
│   │   ├── ChatMessages.vue    # Grid de mensajes (prompts + respuestas)
│   │   ├── ChatPrompt.vue      # Burbuja de prompt del usuario
│   │   ├── ChatResponse.vue    # Tarjeta de respuesta de un bot
│   │   └── ChatThread.vue      # Hilo de seguimiento (reply a un bot)
│   ├── Footer/
│   │   ├── FooterBar.vue       # Barra inferior: textarea + bots seleccionados
│   │   ├── BotLogo.vue         # Icono de bot
│   │   └── BotsMenu.vue        # Menú para añadir/quitar bots
│   ├── ChatDrawer/             # Panel lateral de chats
│   ├── BotSettings/            # Configuraciones por proveedor de IA
│   ├── ChatAction.vue          # ★ CLAVE — Acción sobre respuestas seleccionadas
│   ├── ChatSetting.vue         # Configuración general del chat
│   ├── SettingsModal.vue       # Modal de ajustes
│   ├── PromptModal.vue         # Gestión de prompts guardados
│   └── ProxySetting.vue        # Configuración de proxy
├── store/
│   ├── index.js            # ★ Store Vuex principal (estado, mutations, actions)
│   ├── db.js               # Instancia Dexie (IndexedDB)
│   ├── chats.js            # CRUD de chats
│   ├── messages.js         # CRUD de mensajes
│   ├── threads.js          # CRUD de hilos
│   ├── queue.js            # Cola de actualización con debounce
│   └── migration.js        # Migraciones de datos
├── i18n/                   # Traducciones (EN, ES, ZH, DE, FR, etc.)
├── prompts/                # Biblioteca de prompts predefinidos (EN, ES, ZH)
├── helpers/
│   ├── scroll-helper.js    # Auto-scroll
│   └── template-helper.js  # Interpolación de templates para acciones
└── composables/
    └── matomo.js           # Analytics
```

---

## 3. Flujo de Datos: Del Prompt a la Respuesta

### 3.1 Envío del prompt

```
FooterBar.vue → sendPromptToBots()
  ↓
store.dispatch("sendPrompt", { prompt, bots })
  ↓
Para cada bot en bots[]:
  1. Crea un mensaje tipo "response" en Dexie (vacío, con className del bot)
  2. Llama bot.sendPrompt(prompt, onUpdateResponse, callbackParam)
     ↓
     Bot.sendPrompt() → verifica disponibilidad → llama _sendPrompt()
       ↓
       LangChainBot._sendPrompt() → construye BufferMemory con historial
         → llama model.call(messages) con callback de streaming
         → cada token: onUpdateResponse(callbackParam, {content, done: false})
         → al terminar: onUpdateResponse(callbackParam, {content, done: true})
```

### 3.2 Actualización de la UI

```
onUpdateResponse()
  ↓
dispatch("updateMessage") → empuja a messageQueue
  ↓
Queue.processQueue() (cada 100ms por defecto)
  → Merge actualizaciones pendientes por index
  → bulkUpdate() en Dexie
  ↓
Dexie liveQuery en ChatMessages.vue detecta cambio
  → Re-renderiza ChatResponse.vue con contenido actualizado
```

**Patrón clave**: La UI es reactiva a Dexie, no al store Vuex directamente para mensajes. Vuex maneja configuración y estado global, Dexie maneja datos de chat.

---

## 4. Sistema de Bots — Arquitectura

### 4.1 Jerarquía de clases

```
Bot (clase base abstracta)
├── LangChainBot (bots vía API + LangChain)
│   ├── OpenAIAPIBot → OpenAIAPI4oBot, OpenAIAPIo3Bot, etc.
│   ├── ClaudeAPIBot → ClaudeAPISonnetBot, ClaudeAPIOpusBot, etc.
│   ├── GeminiAPIBot → Gemini20FlashAPIBot, etc.
│   ├── GroqAPIBot → Llama4ScoutGroqAPIBot, etc.
│   ├── CohereAPIBot → CohereAPICommandRBot, etc.
│   └── Grok2APIBot, Grok3APIBot, etc.
└── Bots web (scraping directo, sin LangChain)
    ├── ChatGPT4Bot (web)
    ├── PerplexityBot
    ├── ClaudeAIBot (web)
    ├── BingChatBot
    ├── QianWenBot, SkyWorkBot, KimiBot, etc.
```

### 4.2 Patrón para añadir un bot API

Extremadamente simple — solo 3 propiedades estáticas:

```javascript
// src/bots/openai/OpenAIAPI4oBot.js
import OpenAIAPIBot from "./OpenAIAPIBot";

export default class OpenAIAPI4oBot extends OpenAIAPIBot {
  static _className = "OpenAIAPI4oBot";
  static _logoFilename = "openai-4o-logo.png";
  static _isDarkLogo = true;
  static _model = "gpt-4o";
}
```

Luego se registra en `src/bots/index.js`:
```javascript
const all = [
  ...
  OpenAIAPI4oBot.getInstance(),
  ...
];
```

### 4.3 Propiedades de Bot.js

| Propiedad | Uso |
|---|---|
| `_brandId` | Identificador para i18n |
| `_className` | Identificador único de la clase |
| `_model` | Nombre del modelo (ej: "gpt-4o") |
| `_logoFilename` | Fichero del logo en `public/bots/` |
| `_loginUrl` | URL de login para bots web |
| `_lock` | AsyncLock para serializar peticiones |
| `_settingsComponent` | Componente Vue de configuración |
| `_outputFormat` | "markdown" o "html" |
| `_isAvailable` | Estado de disponibilidad |

### 4.4 Métodos clave de Bot.js

| Método | Descripción |
|---|---|
| `sendPrompt()` | Público — verifica disponibilidad, gestiona lock, llama `_sendPrompt()` |
| `_sendPrompt()` | Abstracto — cada bot implementa su lógica |
| `_checkAvailability()` | Abstracto — verifica login/API key |
| `createChatContext()` | Crea contexto de conversación (historial) |
| `getChatContext()` | Recupera contexto del store |
| `setChatContext()` | Guarda contexto en el store |

---

## 5. Sistema de Acciones (ChatAction) — ★ Base para Orquestación

**Esto es CLAVE para ÁgorAI.** ChatALL ya tiene un mecanismo primitivo de orquestación:

### Flujo actual:
1. El usuario selecciona respuestas de diferentes bots (checkbox en cada tarjeta)
2. Aparece una barra de acciones en el header
3. Cada acción tiene: `prefix` + `template` + `suffix`
4. El `template` se interpola con `{botName}` y `{botResponse}` por cada respuesta seleccionada
5. Se genera un mega-prompt combinando todo
6. Se abre `ChatAction.vue` donde puedes:
   - Editar el prompt generado
   - Elegir qué bots lo reciben
   - Enviar a un chat nuevo o al actual

### Ejemplo de acción existente (Summarize):
```javascript
{
  name: "Summarize 1",
  prefix: "Summarize the data below in a markdown table...",
  template: "<resp>\n  <name>{botName}</name>\n  <value>{botResponse}</value>\n</resp>",
  suffix: "Give me the best response.",
}
```

**Esto es exactamente el 30% de lo que necesita ÁgorAI.** Ya existe la selección de respuestas y la re-inyección a otros bots. Falta: automatización, iteraciones múltiples, y flujos configurables.

---

## 6. Sistema de Hilos (Threads)

ChatALL ya soporta "reply" a una respuesta específica de un bot:

- Cada `ChatResponse.vue` tiene un botón de reply
- Al pulsar, aparece un textarea inline
- `sendPromptInThread()` envía solo a ese bot, manteniendo contexto
- Los threads se almacenan en tabla separada de Dexie

**Para ÁgorAI**: Este sistema se puede extender para soportar "cadenas de orquestación" donde cada paso es un thread que alimenta al siguiente.

---

## 7. Persistencia

| Dato | Almacenamiento | Tecnología |
|---|---|---|
| Configuración global (API keys, idioma, tema, columnas) | Vuex + localStorage via vuex-persist | localForage |
| Chats | IndexedDB | Dexie |
| Mensajes | IndexedDB | Dexie |
| Threads | IndexedDB | Dexie |
| Proxy | Fichero JSON | Electron fs |

**Esquema Dexie:**
```javascript
db.version(1).stores({
  chats: "index, title, modifiedTime, selectedTime",
  messages: "index, chatIndex, createdTime, modifiedTime",
  threads: "index, chatIndex, messageIndex, createdTime, modifiedTime",
});
```

---

## 8. Build y Distribución

- **electron-builder** para empaquetar
- **Windows**: instalador NSIS (`.exe`)
- **macOS**: `.dmg` (arm64 + x64), firmado y notarizado
- **Linux**: `.AppImage` + `.deb`
- **Auto-update**: `update-electron-app`
- Config en `vue.config.js` → `pluginOptions.electronBuilder`

---

## 9. Puntos de Extensión — Dónde Tocar para ÁgorAI

### ✅ Fácil de extender (modular)

| Área | Por qué |
|---|---|
| **Añadir bots** | Patrón de herencia limpio: crear clase, 3 propiedades, registrar en index.js |
| **Acciones** | `ChatAction.vue` ya soporta template + re-inyección, solo hay que expandirlo |
| **Store** | Vuex bien estructurado, fácil añadir nuevos estados y mutations |
| **Persistencia** | Dexie permite añadir tablas fácilmente (para sesiones de orquestación) |
| **i18n** | Sistema multiidioma completo, fácil añadir strings |

### ⚠️ Requiere refactoring

| Área | Por qué |
|---|---|
| **ChatMessages.vue** | El grid está pensado para "un prompt → N respuestas planas". Hay que añadir visualización de rondas/iteraciones |
| **FooterBar.vue** | Muy acoplado al flujo "escribir → enviar a bots seleccionados". Necesita modo orquestador |
| **Store/index.js** | 580+ líneas, demasiadas mutations directas. Conviene modularizar para orquestación |

### 🔴 Más complejo

| Área | Por qué |
|---|---|
| **Bots web (scraping)** | Dependen de reverse engineering de interfaces web, frágiles y difíciles de mantener |
| **background.js** | Gestión de proxy y ventanas embebidas para bots web. Si se priorizan bots API, esto pierde relevancia |

---

## 10. Propuesta de Módulos para ÁgorAI

### Fase 1 — Motor de Orquestación (Core)

#### Módulo 1.1: `OrchestrationEngine`
**Ubicación**: `src/orchestration/engine.js`
**Función**: Motor central que gestiona sesiones de orquestación iterativa
- Define una "sesión" como una secuencia de rondas
- Cada ronda tiene: prompt de entrada, bots destino, respuestas obtenidas
- Permite avanzar a la siguiente ronda re-inyectando respuestas seleccionadas
- Estado: idle → running → waiting_selection → running → ... → completed

```
Nueva tabla Dexie: orchestrations
  index, chatIndex, rounds[], status, createdTime, config
```

#### Módulo 1.2: `RoundManager`
**Ubicación**: `src/orchestration/round.js`
**Función**: Gestiona una ronda individual
- Envía prompt a N bots en paralelo (reutiliza sendPrompt existente)
- Espera a que todos respondan (o timeout)
- Presenta respuestas para selección del usuario
- Genera el prompt de la siguiente ronda con las respuestas seleccionadas

#### Módulo 1.3: `PromptComposer`
**Ubicación**: `src/orchestration/composer.js`
**Función**: Genera prompts compuestos para rondas intermedias
- Extiende el template-helper.js existente
- Templates configurables: debate, refinamiento, síntesis, crítica
- Variables: `{previousResponses}`, `{selectedResponse}`, `{roundNumber}`, `{originalPrompt}`

### Fase 2 — UI de Orquestación

#### Módulo 2.1: `OrchestrationView.vue`
**Ubicación**: `src/components/Orchestration/OrchestrationView.vue`
**Función**: Vista principal del modo orquestador
- Timeline visual de rondas (vertical, tipo flujo)
- Cada ronda muestra: prompt enviado, bots usados, respuestas
- Controles: "Siguiente ronda", "Seleccionar respuestas", "Finalizar"
- Cambio de vista: modo clásico (ChatALL) ↔ modo orquestador (ÁgorAI)

#### Módulo 2.2: `RoundCard.vue`
**Ubicación**: `src/components/Orchestration/RoundCard.vue`
**Función**: Tarjeta visual de una ronda
- Muestra respuestas de cada bot en esa ronda
- Checkboxes para seleccionar qué respuestas pasan a la siguiente ronda
- Indicador de estado (en progreso, completada, pendiente)

#### Módulo 2.3: `OrchestrationFooter.vue`
**Función**: Footer alternativo para modo orquestación
- Reemplaza FooterBar cuando estás en modo orquestador
- Controles de ronda: configurar bots, template, enviar siguiente ronda
- Botón "Finalizar y exportar"

### Fase 3 — Integración MCP

#### Módulo 3.1: `MCPManager`
**Ubicación**: `src/mcp/manager.js`
**Función**: Gestor de conexiones MCP
- Descubrimiento y registro de servidores MCP
- Gestión del ciclo de vida (connect, disconnect, health check)
- Configuración por servidor (URL, auth, capabilities)

#### Módulo 3.2: `MCPBot` (nuevo tipo de bot)
**Ubicación**: `src/bots/mcp/MCPBot.js`
**Función**: Bot que actúa como bridge a un servidor MCP
- Hereda de Bot.js
- En vez de llamar a una API de LLM, ejecuta herramientas MCP
- Permite que durante la orquestación un "bot" sea realmente una herramienta
- Ejemplo: SAPladdin como "bot" que consulta datos SAP en una ronda

#### Módulo 3.3: `MCPToolPanel.vue`
**Función**: Panel de herramientas MCP disponibles
- Lista servidores MCP conectados y sus tools
- Permite invocar tools manualmente durante una orquestación
- Muestra resultados que pueden inyectarse como contexto

### Fase 4 — Flujos Predefinidos y Exportación

#### Módulo 4.1: `FlowTemplates`
**Ubicación**: `src/orchestration/templates/`
**Función**: Plantillas de flujos de orquestación
- **Debate**: Modelo A opina → Modelo B contraargumenta → Modelo C sintetiza
- **Refinamiento**: Modelo A genera → Modelo B critica → Modelo A mejora
- **Fact-check**: Modelo A afirma → Modelos B,C,D verifican → Síntesis
- **Code Review**: Modelo A genera código → Modelo B revisa → Modelo A corrige

#### Módulo 4.2: `SessionExporter`
**Ubicación**: `src/orchestration/exporter.js`
**Función**: Exporta sesiones completas
- Markdown con toda la cadena de orquestación
- JSON estructurado
- Integración con Joplin MCP para guardar directamente en notas

### Fase 5 — Resiliencia y Framework Personal

#### Módulo 5.1: `BotHealthMonitor`
**Función**: Monitorización de salud de bots
- Detecta si un bot/API está caído o con errores
- Sugerencia automática de alternativas
- Fallback automático configurable (si Claude falla, enviar a GPT)

#### Módulo 5.2: `WorkspaceManager`
**Función**: Marco de trabajo personal
- Perfiles de trabajo: "Investigación", "Código", "Contenido"
- Cada perfil preselecciona bots, templates y MCPs
- Historial de sesiones de orquestación por proyecto

---

## 11. Prioridad de Implementación Sugerida

| Prioridad | Módulo | Esfuerzo | Dependencias |
|---|---|---|---|
| 🔴 P0 | 1.1 OrchestrationEngine | Alto | Nueva tabla Dexie, store |
| 🔴 P0 | 1.2 RoundManager | Medio | Reutiliza sendPrompt |
| 🔴 P0 | 1.3 PromptComposer | Bajo | Extiende template-helper |
| 🟡 P1 | 2.1 OrchestrationView | Alto | Motor completo |
| 🟡 P1 | 2.2 RoundCard | Medio | Extiende ChatResponse |
| 🟡 P1 | 2.3 OrchestrationFooter | Medio | Extiende FooterBar |
| 🟢 P2 | 3.1 MCPManager | Alto | SDK MCP |
| 🟢 P2 | 3.2 MCPBot | Medio | MCPManager |
| 🟢 P2 | 3.3 MCPToolPanel | Medio | MCPManager |
| 🔵 P3 | 4.1 FlowTemplates | Bajo | Motor completo |
| 🔵 P3 | 4.2 SessionExporter | Bajo | Motor completo |
| ⚪ P4 | 5.1 BotHealthMonitor | Medio | - |
| ⚪ P4 | 5.2 WorkspaceManager | Alto | Todo lo anterior |

---

## 12. Archivos Clave a Modificar

| Archivo | Cambio |
|---|---|
| `src/store/db.js` | Añadir tabla `orchestrations` |
| `src/store/index.js` | Nuevos states, mutations y actions para orquestación |
| `src/App.vue` | Toggle entre modo clásico y modo orquestador |
| `src/bots/index.js` | Registrar MCPBots |
| `src/components/Messages/ChatMessages.vue` | Soportar vista de rondas |
| `vue.config.js` | Actualizar productName a "ÁgorAI" |
| `package.json` | Actualizar nombre, añadir deps MCP |

---

*Análisis realizado desde Perplexity Computer — 01/04/2026*
