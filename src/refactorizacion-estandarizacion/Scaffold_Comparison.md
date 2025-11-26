# Comparativa de Estructura de Proyecto (Scaffold): As-Is vs To-Be

Este documento ilustra la evolución de la estructura de carpetas del módulo `server/workflow` para adoptar la Arquitectura Hexagonal. El objetivo es mejorar la mantenibilidad, facilitar la incorporación de nuevos desarrolladores y soportar múltiples flujos de modernización (Discovery, Rebuild, etc.) de forma estandarizada.

## 1. Estructura Actual (As-Is)

La estructura actual mezcla responsabilidades. Los "Agentes" son a la vez lógica de negocio y clientes HTTP. Los procesos de modernización están agrupados por cliente (MABA, TIBCO) en lugar de por tipo de flujo.

```text
server/workflow/src/
├── application/
│   ├── MABA/                          <-- Mezcla orquestación y definiciones de agentes
│   │   ├── agents/                    <-- Implementaciones acopladas a Coding (extienden BaseCodingTool)
│   │   │   ├── LegacyAgent.ts
│   │   │   ├── BIANAgent.ts
│   │   │   ├── ARQAgent.ts
│   │   │   └── ...
│   │   ├── modernizationLeadDiscoveryProcess.ts  <-- Orquestador Discovery
│   │   ├── modernizationLeadRebuildProcess.ts    <-- Orquestador Rebuild
│   │   └── agentRunner.ts             <-- Helper de ejecución específico
│   └── TIBCO/                         <-- Otro cliente con estructura similar
├── infrastructure/
│   ├── CodingConnectorTool.ts         <-- Cliente HTTP específico de Coding
│   ├── StreamService.ts               <-- Gestión de SSE
│   └── ...
├── tools/
│   ├── BaseCodingTool.ts              <-- Clase base que acopla todo a Coding
│   └── ...
├── types/
│   └── agents.ts                      <-- Tipos mezclados (Coding + Negocio)
└── ...
```

### Puntos de Dolor
*   **Difícil de navegar:** ¿Dónde está la lógica de negocio de Legacy? En `agents/LegacyAgent.ts`, pero está llena de código de infraestructura.
*   **Difícil de escalar:** Añadir un nuevo flujo (ej. "Rework") implica copiar y pegar mucha lógica de orquestación.
*   **Acoplamiento:** `application/MABA/agents` depende directamente de `tools/BaseCodingTool`.

---

## 2. Estructura Propuesta (To-Be)

La nueva estructura separa claramente **Dominio** (Qué hacemos), **Infraestructura** (Con qué herramientas externas) y **Aplicación** (Cómo coordinamos los flujos).

```text
server/workflow/src/
├── domain/                            <-- EL NÚCLEO (Puro TypeScript, sin dependencias externas)
│   ├── ports/                         <-- Contratos (Interfaces)
│   │   ├── IAgentConnector.ts         <-- Contrato para cualquier IA (Coding, Azure, etc.)
│   │   └── IStreamService.ts          <-- Contrato para streams de eventos
│   ├── types/                         <-- Lenguaje Ubicuo (Nuevo Enfoque)
│   │   ├── StandardRequest.ts         <-- { TYPE_AGENT, LEGACY_SOURCE, ... }
│   │   └── StandardResponse.ts        <-- { DOC_FUNCTIONAL, DIAGRAMS_TEXT, ... }
│   └── services/                      <-- Lógica de Negocio Pura (Servicios de Dominio)
│       ├── LegacyAnalysisService.ts   <-- "Sabe" qué pedir, no a quién
│       ├── BianArchitectureService.ts
│       └── EngineeringService.ts
│
├── infrastructure/                    <-- LOS ADAPTADORES (Implementación Técnica)
│   ├── adapters/
│   │   ├── coding/                    <-- Implementación específica para "Coding"
│   │   │   ├── CodingAgentAdapter.ts  <-- Traduce StandardRequest -> Coding JSON
│   │   │   └── CodingStreamAdapter.ts
│   │   └── azure/                     <-- (Futuro) Implementación para Azure AI
│   │       └── AzureOpenAIAdapter.ts
│   └── config/                        <-- Inyección de dependencias
│       └── ContainerConfig.ts         <-- Decide qué adaptador usar (Coding vs Azure)
│
├── application/                       <-- LA ORQUESTACIÓN (Casos de Uso / Flujos)
│   ├── workflows/                     <-- Los diferentes caminos de modernización
│   │   ├── discovery/
│   │   │   └── DiscoveryWorkflow.ts   <-- (Antes modernizationLeadDiscoveryProcess)
│   │   ├── rebuild/
│   │   │   └── RebuildWorkflow.ts     <-- (Antes modernizationLeadRebuildProcess)
│   │   └── rework/
│   │       └── ReworkWorkflow.ts
│   └── dtos/                          <-- Objetos de transferencia para la API REST
│
└── shared/                            <-- Utilidades transversales
    └── utils/
```

### Ventajas para el Equipo
1.  **Estandarización:** Un desarrollador nuevo sabe que si quiere ver *cómo* se analiza el código Legacy, va a `domain/services/LegacyAnalysisService`. Si quiere ver *cómo* se conecta con la API, va a `infrastructure/adapters`.
2.  **Soporte a Múltiples Procesos:** Los `workflows` (Discovery, Rebuild) son orquestadores que simplemente llaman a los servicios de dominio (`LegacyAnalysisService`, `BianArchitectureService`) en diferente orden. Reutilizan la lógica sin duplicarla.
3.  **Flexibilidad de Agentes:**
    *   El `DiscoveryWorkflow` llama a `LegacyAnalysisService`.
    *   El `LegacyAnalysisService` usa `IAgentConnector`.
    *   En tiempo de ejecución, inyectamos `CodingAgentAdapter`.
    *   Si mañana queremos probar GPT-4 para Legacy, inyectamos `AzureOpenAIAdapter` y el workflow no cambia ni una línea.

## 3. Ejemplo de Flujo en la Nueva Estructura

**Escenario:** Ejecutar proceso de Discovery.

1.  **Entrada:** `application/workflows/discovery/DiscoveryWorkflow.ts` recibe la petición.
2.  **Paso 1 (Legacy):** El workflow llama a `domain/services/LegacyAnalysisService.analyze(sourceCode)`.
3.  **Dominio:** El servicio crea un `StandardRequest` con `TYPE_AGENT='legacy'` y `LEGACY_SOURCE=sourceCode`.
4.  **Puerto:** El servicio llama a `IAgentConnector.execute(request)`.
5.  **Infraestructura:** `CodingAgentAdapter` intercepta la llamada:
    *   Traduce `LEGACY_SOURCE` a `input` (formato Coding).
    *   Hace POST a la API de Coding.
    *   Espera el Stream.
    *   Traduce `functionalDetail` a `DOC_FUNCTIONAL`.
6.  **Retorno:** El servicio recibe un `StandardResponse` limpio y se lo devuelve al workflow.
7.  **Paso 2 (BIAN):** El workflow toma `DOC_FUNCTIONAL` del paso anterior y llama a `domain/services/BianArchitectureService.design(...)`.

Este ciclo se repite, manteniendo cada capa limpia y enfocada en su responsabilidad.
