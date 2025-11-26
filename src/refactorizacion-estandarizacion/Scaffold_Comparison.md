# Reestructuración del Proyecto: Comparativa y Propuesta

Propuesta de reorganización del módulo `server/workflow` para alinearlo con la Arquitectura Hexagonal. El objetivo principal es desacoplar la lógica de negocio de la infraestructura, facilitando la mantenibilidad y la incorporación de nuevos flujos de modernización.

## 1. Situación Actual (As-Is)

Actualmente, la estructura mezcla responsabilidades. Las clases de "Agentes" contienen tanto lógica de negocio como implementación de clientes HTTP. Además, la agrupación por cliente (MABA, TIBCO) dificulta la reutilización de lógica común.

```text
server/workflow/src/
├── application/
│   ├── MABA/                          <-- Mezcla orquestación y definición de agentes
│   │   ├── agents/                    <-- Implementaciones acopladas a Coding (herencia de BaseCodingTool)
│   │   │   ├── LegacyAgent.ts
│   │   │   ├── BIANAgent.ts
│   │   │   └── ...
│   │   ├── modernizationLeadDiscoveryProcess.ts
│   │   └── ...
│   └── TIBCO/
├── infrastructure/
│   ├── CodingConnectorTool.ts         <-- Cliente HTTP específico
│   └── ...
├── tools/
│   ├── BaseCodingTool.ts              <-- Clase base que genera el acoplamiento
│   └── ...
└── ...
```

### Problemas Detectados
*   **Navegación compleja:** La lógica de negocio está dispersa y mezclada con código de infraestructura.
*   **Escalabilidad limitada:** Añadir nuevos flujos (ej. "Rework") requiere duplicar lógica de orquestación.
*   **Alto Acoplamiento:** Dependencia directa entre `application` y `tools/BaseCodingTool`.

---

## 2. Nueva Organización (To-Be)

La propuesta separa claramente **Dominio** (Reglas de negocio), **Infraestructura** (Adaptadores externos) y **Aplicación** (Orquestación).

```text
server/workflow/src/
├── domain/                            <-- NÚCLEO (TypeScript puro, sin dependencias)
│   ├── ports/                         <-- Interfaces / Contratos
│   │   ├── IAgentConnector.ts         <-- Contrato agnóstico para IAs
│   │   └── IStreamService.ts
│   ├── types/                         <-- Definiciones del Dominio (Lenguaje Ubicuo)
│   │   ├── StandardRequest.ts
│   │   └── StandardResponse.ts
│   └── services/                      <-- Lógica de Negocio
│       ├── LegacyAnalysisService.ts
│       ├── BianArchitectureService.ts
│       └── ...
│
├── infrastructure/                    <-- ADAPTADORES
│   ├── adapters/
│   │   ├── coding/                    <-- Implementación para "Coding"
│   │   │   ├── CodingAgentAdapter.ts
│   │   │   └── ...
│   │   └── azure/                     <-- Futura implementación Azure
│   └── config/
│       └── ContainerConfig.ts         <-- Inyección de dependencias
│
├── application/                       <-- ORQUESTACIÓN
│   ├── workflows/                     <-- Flujos de modernización
│   │   ├── discovery/
│   │   │   └── DiscoveryWorkflow.ts
│   │   ├── rebuild/
│   │   │   └── RebuildWorkflow.ts
│   │   └── ...
│   └── dtos/
│
└── shared/
```

### Mejoras Clave
1.  **Claridad:** Separación estricta entre el *qué* (Dominio) y el *cómo* (Infraestructura).
2.  **Reutilización:** Los `workflows` orquestan servicios de dominio reutilizables, evitando duplicidad.
3.  **Flexibilidad:** Permite inyectar diferentes adaptadores (Coding, Azure) sin modificar la lógica de negocio.

## 3. Flujo de Ejecución (Ejemplo Discovery)

1.  **Entrada:** `DiscoveryWorkflow` recibe la petición.
2.  **Ejecución:** Llama a `LegacyAnalysisService.analyze(sourceCode)`.
3.  **Dominio:** El servicio genera un `StandardRequest` y llama al puerto `IAgentConnector`.
4.  **Infraestructura:** `CodingAgentAdapter` intercepta la llamada, traduce al formato de Coding, ejecuta la petición HTTP y espera el Stream.
5.  **Retorno:** Se devuelve un `StandardResponse` normalizado al workflow.
6.  **Siguiente Paso:** El workflow usa la respuesta para invocar a `BianArchitectureService`.

Esta estructura garantiza que cada capa tenga una responsabilidad única y aislada.
