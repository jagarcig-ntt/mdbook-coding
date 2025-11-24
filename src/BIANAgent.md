# BIAN AGENT — CODING

## 1. Propósito del flujo
| Objetivo |
|----------|
| Clasificar programas/JCL Legacy en SD BIAN a traves de su documentación funcional y técnica, generar diagramas, mapear Use Cases a Business Scenarios y construir el landscape BIAN por cliente. |

---

## 2. Entradas del proceso
| Variable | Descripción |
|----------|-------------|
| `inputFunc` | Documentación funcional generada por el Legacy Agent |
| `inputTech` | Documentación técnica generada por el Legacy Agent |
| `client` | Identificador del cliente |

---

## 3. Fases del flujo

### 3.1 Contexto BIAN
| Dataset | Uso |
|---------|-----|
| `bian-service-landscape-12-v-2:1.0.0` | Landscape BIAN (Business Area, Business Domain, Service Domain) |
| `use-case-datasets:1.0.0` | Business Scenarios |

---

### 3.2 Generación con IA
| Artefacto | Workflow | Prompt |
|-----------|----------|--------|
| Documentación BIAN | `call-ai-v-2:1.0.3` | `bian-agent-prompt-v-2:1.0.3` |
| Mapeo de escenarios | `call-ai-v-2:1.0.3` | `bian-business-scenario-mapping-prompt:1.0.0` |

---

### 3.3 Postproceso
| Snippet | Función |
|---------|---------|
| `bian-postprocess-ai-documentation-generate:1.0.2` | Extrae ProgramName, BIANDocumentationName, MermaidDiagram y ServiceDomain |

---

### 3.4 Obtención de Use Cases
| Acción | Workflow |
|--------|----------|
| Recuperación de Use Cases del Legacy Agent | `dataset-search:1.0.1` |

---

### 3.5 Persistencia
| Artefacto | Dataset actualizado |
|-----------|---------------------|
| Documentación BIAN | `bian-dataset-v-2:1.0.0` |
| Diagrama BIAN | `bian-diagram-dataset-v-2:1.0.0` |
| Landscape cliente | `bian-client-diagram-dataset-v-2:1.0.0` |

---

### 3.6 Recuperación de URLs
| Tipo | Dataset |
|------|---------|
| API del Service Domain | `api-bian-12-v-2:latest` |
| Home del SD | `bian-service-domain-home:latest` |
| SD Overview | `bian-service-domain-sd-overview-url:latest` |

---

### 3.7 Limpieza
| Acción | Workflow |
|--------|----------|
| Eliminación de variables internas | `MapValues clean_workflow` |

---

## 4. Flujo lógico (resumen)
| Paso | Descripción |
|------|-------------|
| 1 | Leer Landscape y Business Scenarios |
| 2 | Generar documentación BIAN |
| 3 | Postprocesar salida IA |
| 4 | Buscar Use Cases legacy |
| 5 | Mapear Use Cases ↔ Scenarios |
| 6 | Persistir documentación/diagramas |
| 7 | Agregar landscape cliente |
| 8 | Extraer URLs |
| 9 | Limpiar y emitir JSON final |

---

## 5. Salida del flujo
El flujo produce y normaliza las siguientes variables, que constituyen el output final:

| Variable | Descripción |
|----------|-------------|
| `ProgramName` | Nombre del programa procesado |
| `BIANDocumentation` | Documentación BIAN generada |
| `BIANDocumentationName` | Nombre asignado a la documentación BIAN |
| `MermaidDiagram` | Diagrama Mermaid para el Service Domain |
| `MermaidDiagramName` | Nombre del diagrama Mermaid |
| `ServiceDomain` | Service Domain identificado para el programa |
| `ServiceDomainSinEspacios` | SD formateado para URLs/repositorios |
| `URL_API_BIAN` | URL de la API del Service Domain |
| `URL_HOME_BIAN` | URL de la página principal del SD |
| `URL_SD_Overview_BIAN` | URL del SD Overview |
| `client_diagram` | Landscape BIAN consolidado para el cliente |
| `bian_output` | Respuesta IA cruda antes de postproceso |
| `bian_mapping_scenarios` | Mapeo de Use Cases ↔ Business Scenarios |

