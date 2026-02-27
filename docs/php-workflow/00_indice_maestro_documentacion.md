# 📚 Índice Maestro - Documentación Completa de php-workflow

**Estado**: ✅ 100% Completado (16 documentos, 14 grupos, 50+ temas, 180+ subtemas)

**Última actualización**: 2026-02-23

**Total de palabras**: ~110,000-130,000

---

## 🎯 Guía de Navegación Rápida

### Por Nivel de Experiencia

**🟢 Principiante - Comienza aquí:**
1. [01: Creación de Workflows](#doc-01-creación-de-workflows)
2. [02: Definición de Steps](#doc-02-definición-de-steps)
3. [10: Best Practices](#doc-10-best-practices-y-patrones)

**🟡 Intermedio - Aprende características:**
4. [03: Logging](#doc-03-logging-y-registros)
5. [04: Control de Flujo](#doc-04-control-de-flujo-avanzado)
6. [07: Loops](#doc-07-loops-y-loopcontrol)
7. [11: Testing](#doc-11-testing-strategies)

**🔴 Avanzado - Domina patrones:**
8. [05: Workflows Anidados](#doc-05-workflows-anidados)
9. [06: Middleware](#doc-06-middleware-y-extensibilidad)
10. [08: Dependencias](#doc-08-dependencias-entre-steps)
11. [15: Advanced Topics](#doc-15-advanced-topics)

**📖 Referencia - Consulta rápida:**
12. [13: API Reference](#doc-13-api-reference) ← MEJOR PARA LOOKUPS
13. [14: Debugging](#doc-14-debugging-y-troubleshooting)

**🚀 Integración - Casos reales:**
14. [12: Integración y Casos de Uso](#doc-12-integración-y-casos-de-uso)
15. [16: Optional Features](#doc-16-optional-features)

---

## 📋 Tabla de Contenidos Completa

### ✅ GRUPO 0: Baseline

#### DOC-01: Creación de Workflows

**Path**: [01_creacion_workflow_php-workflow.md](01_creacion_workflow_php-workflow.md)

**Secciones**:
- 1. Qué es un Workflow
- 2. Constructor Workflow
- 3. Ciclo de Vida Completo
- 4. Builder Pattern
- 5. Method Chaining
- 6. Etapas (Stages)
- 7. Workflow Básico
- 8. Manejo de Excepciones
- 9. Container Compartido
- 10. Primer Workflow Funcional

**Mejor para**: Entender estructura básica, primer contacto

---

#### DOC-02: Definición de Steps

**Path**: [02_definicion_pasos_workflow_php-workflow.md](02_definicion_pasos_workflow_php-workflow.md)

**Secciones**:
- 1. WorkflowStep Interface
- 2. Método run()
- 3. Método getDescription()
- 4. Ciclo de Vida de un Step
- 5. Container y Control
- 6. Tipos de Steps
- 7. Step con Lógica
- 8. Manejo de Errores en Steps
- 9. Validación en Steps
- 10. Steps Reutilizables
- 11. Dependencias Simples
- 12. Logging en Steps
- 13. Context en Steps
- 14. Cancel y Skip
- 15. Exception Handling
- 16. Async en Steps
- 17. Testing Steps
- 18. Step Composition

**Mejor para**: Crear steps, entender interfaz WorkflowStep

---

### ✅ GRUPO 1: Logging y Registros

#### DOC-03: Logging y Registros

**Path**: [03_logging_registros_workflow_php-workflow.md](03_logging_registros_workflow_php-workflow.md)

**Secciones**:
- 1. Sistema de ExecutionLog
- 2. Estructura interna de logs
- 3. StepInfo y Context
- 4. Warnings Management
- 5. OutputFormatters
- 6. StringLog Formatter
- 7. Custom Formatters
- 8. GraphViz Integration
- 9. Timing y Performance
- 10. Describable Interface
- 11. Log Inspection
- 12. Debug Output
- 13. Contextual Information
- 14. Message Formatting
- 15. Production Logging
- 16. Log Aggregation
- 17. Performance Impact
- 18. Best Practices
- 19. Anti-patterns

**Mejor para**: Entender logs, debugging, monitoring

---

### ✅ GRUPO 2: Control de Flujo Avanzado

#### DOC-04: Control de Flujo Avanzado

**Path**: [04_control_flujo_avanzado_workflow_php-workflow.md](04_control_flujo_avanzado_workflow_php-workflow.md)

**Secciones**:
- 1. Excepciones de Control
- 2. SkipStepException
- 3. FailStepException
- 4. FailWorkflowException
- 5. SkipWorkflowException
- 6. ContinueException
- 7. BreakException
- 8. Jerarquía de Excepciones
- 9. Etapa OnSuccess
- 10. Etapa OnError
- 11. Etapa After
- 12. Punto de No Retorno
- 13. Flujo de Ejecución
- 14. Error Recovery
- 15. Transaccional Patterns
- 16. Rollback Strategies
- 17. Conditional Execution
- 18. State Management
- 19. Complex Flows

**Mejor para**: Control de flujo complejo, manejo de errores avanzado

---

### ✅ GRUPO 3: Loops y LoopControl

#### DOC-07: Loops y LoopControl

**Path**: [07_loops_loopcontrol_workflow_php-workflow.md](07_loops_loopcontrol_workflow_php-workflow.md)

**Secciones**:
- 1. Concepto de Loop dentro de Workflow
- 2. LoopControl Interface
- 3. Contador Loop
- 4. Colección Loop
- 5. Conditional Loop
- 6. Backoff Loop
- 7. executeNextIteration()
- 8. Continue en Loops
- 9. Break en Loops
- 10. skipStep() en Loops
- 11. Exception Handling
- 12. continueOnError Flag
- 13. Loop Logging
- 14. Nested Loops
- 15. Loops en NestedWorkflow
- 16. Loops en diferentes Etapas
- 17. Performance
- 18. Debugging Loops
- 19. Best Practices
- 20. Anti-patterns
- 21. Real Examples
- 22. Timing y Limits

**Mejor para**: Iteración, procesamiento en lote, patrones de repetición

---

### ✅ GRUPO 4: Workflows Anidados

#### DOC-05: Workflows Anidados

**Path**: [05_workflows_anidados_workflow_php-workflow.md](05_workflows_anidados_workflow_php-workflow.md)

**Secciones**:
- 1. Concepto de Nesting
- 2. NestedWorkflow Step
- 3. Container Sharing
- 4. NestedContainer Inheritance
- 5. Data Propagation
- 6. Error Propagation
- 7. getNestedWorkflowResult()
- 8. Cascada de Workflows
- 9. Logging en Nesting
- 10. Deep Nesting
- 11. Parallel Workflows
- 12. Container Isolation
- 13. Shared State
- 14. Result Combination
- 15. Exception Handling
- 16. Performance
- 17. Debugging
- 18. Best Practices
- 19. Anti-patterns
- 20. Real Examples

**Mejor para**: Composición de workflows, reutilización, modularidad

---

### ✅ GRUPO 5: Middleware y Extensibilidad

#### DOC-06: Middleware y Extensibilidad

**Path**: [06_middleware_extensibilidad_workflow_php-workflow.md](06_middleware_extensibilidad_workflow_php-workflow.md)

**Secciones**:
- 1. Concepto de Middleware
- 2. ProfileStep Built-in
- 3. WorkflowStepDependencyCheck
- 4. Middleware Callable
- 5. LIFO Wrapping
- 6. Crear Middleware Custom
- 7. Logging Middleware
- 8. Transaction Middleware
- 9. Rate Limiting
- 10. Caching Middleware
- 11. Timing Middleware
- 12. Exception Handling
- 13. Middleware en Bucles
- 14. Middleware en NestedWorkflow
- 15. Performance Impact
- 16. Composition Patterns
- 17. Debugging Middleware
- 18. Anti-patterns
- 19. Real Examples

**Mejor para**: Extensión, cross-cutting concerns, instrumentación

---

### ✅ GRUPO 6: Dependencias Entre Steps

#### DOC-08: Dependencias Entre Steps

**Path**: [08_dependencias_steps_workflow_php-workflow.md](08_dependencias_steps_workflow_php-workflow.md)

**Secciones**:
- 1. Concepto de Dependencias
- 2. StepDependencyInterface
- 3. Requires Attribute (PHP 8+)
- 4. WorkflowStepDependencyCheck
- 5. Tipos Soportados
- 6. Validación Automática
- 7. Error Messages
- 8. Optional Dependencies
- 9. Preconditions
- 10. Custom Dependencies
- 11. Dependency Injection
- 12. Container Integration
- 13. Type Checking
- 14. Runtime Validation
- 15. Error Handling
- 16. Best Practices
- 17. Anti-patterns
- 18. Real Examples
- 19. Testing Dependencies
- 20. Documentation

**Mejor para**: Validación formal, PHP 8+ features, precondiciones

---

### ✅ GRUPO 7: WorkflowResult y Resultados

#### DOC-09: WorkflowResult

**Path**: [09_workflowresult_workflow_php-workflow.md](09_workflowresult_workflow_php-workflow.md)

**Secciones**:
- 1. WorkflowResult Interface
- 2. success() Method
- 3. getException() Method
- 4. getContainer() Method
- 5. getLastStep() Method
- 6. getWarnings() Method
- 7. hasWarnings() Method
- 8. debug() Method
- 9. getWorkflowName() Method
- 10. WorkflowException Wrapping
- 11. Exception Recovery
- 12. Result Inspection
- 13. Data Access
- 14. Warning Analysis
- 15. Success Patterns
- 16. Failure Patterns
- 17. Debugging Results
- 18. Production Usage

**Mejor para**: Acceder resultados, manejo de excepciones, inspección final

---

### ✅ GRUPO 8: Best Practices y Patrones

#### DOC-10: Best Practices y Patrones

**Path**: [10_best_practices_patrones_workflow_php-workflow.md](10_best_practices_patrones_workflow_php-workflow.md)

**Secciones**:
- 1. Step Design Patterns
- 2. Service Layer Pattern
- 3. Factory Pattern
- 4. Repository Pattern
- 5. Domain Events Pattern
- 6. Command Pattern
- 7. Stateful Steps Anti-pattern
- 8. Direct State Access Anti-pattern
- 9. Container Key Conflicts Anti-pattern
- 10. Exception Swallowing Anti-pattern
- 11. Performance Optimization
- 12. Memory Management
- 13. Caching Strategies
- 14. Batch Processing
- 15. Security & Validation
- 16. Data Isolation
- 17. Error Handling Best Practices
- 18. Testing Strategies

**Mejor para**: Arquitectura, decisiones de diseño, anti-patrones

---

### ✅ GRUPO 9: Testing Strategies

#### DOC-11: Testing Strategies

**Path**: [11_testing_strategies_workflow_php-workflow.md](11_testing_strategies_workflow_php-workflow.md)

**Secciones**:
- 1. Testing Philosophy
- 2. Unit Testing Steps
- 3. Integration Testing
- 4. WorkflowTestTrait
- 5. WorkflowSetupTrait
- 6. Mock WorkflowControl
- 7. Mock WorkflowContainer
- 8. Data-Driven Tests
- 9. Testing Loops
- 10. Testing Nested Workflows
- 11. Testing Middleware
- 12. Testing Error Cases
- 13. Testing Validation
- 14. Fixtures & Factories
- 15. Best Practices
- 16. Anti-patterns
- 17. Coverage Goals
- 18. Performance Testing
- 19. Debugging Tests
- 20. CI/CD Integration

**Mejor para**: Testing, quality assurance, confidence building

---

### ✅ GRUPO 10: Integración y Casos de Uso

#### DOC-12: Integración y Casos de Uso

**Path**: [12_integracion_casos_uso_workflow_php-workflow.md](12_integracion_casos_uso_workflow_php-workflow.md)

**Secciones**:
- 1. Standalone Usage
- 2. Dependency Container Setup
- 3. Laravel Integration
- 4. Symfony Integration
- 5. Service Provider Pattern
- 6. Use Case: E-commerce Checkout
- 7. Use Case: User Registration
- 8. Use Case: ETL Pipeline
- 9. Use Case: Approval Workflow
- 10. Use Case: Report Generation
- 11. Error Handling in Production
- 12. Logging & Monitoring
- 13. Performance Tuning
- 14. Database Transactions
- 15. API Wrappers

**Mejor para**: Casos reales, integración con frameworks, producción

---

### ✅ GRUPO 11: API Reference

#### DOC-13: API Reference

**Path**: [13_api_reference_workflow_php-workflow.md](13_api_reference_workflow_php-workflow.md)

**Secciones**:
- 1. Workflow Constructor
- 2. Workflow.prepare()
- 3. Workflow.validate()
- 4. Workflow.before()
- 5. Workflow.process()
- 6. Workflow.onSuccess()
- 7. Workflow.onError()
- 8. Workflow.after()
- 9. Workflow.executeWorkflow()
- 10. WorkflowStep Interface
- 11. WorkflowStep.run()
- 12. WorkflowStep.getDescription()
- 13. WorkflowControl Methods
- 14. WorkflowControl.skipStep()
- 15. WorkflowControl.failStep()
- 16. WorkflowControl.failWorkflow()
- 17. WorkflowControl.skipWorkflow()
- 18. WorkflowControl.continue()
- 19. WorkflowControl.break()
- 20. WorkflowControl.attachStepInfo()
- 21. WorkflowControl.warning()
- 22. WorkflowContainer Methods
- 23. WorkflowContainer.get()
- 24. WorkflowContainer.set()
- 25. WorkflowContainer.has()
- 26. WorkflowContainer.unset()
- 27. WorkflowContainer.keys()
- 28. WorkflowResult Methods
- 29. WorkflowResult.success()
- 30. WorkflowResult.getException()
- 31. WorkflowResult.getContainer()
- 32. WorkflowResult.getLastStep()
- 33. WorkflowResult.getWarnings()
- 34. WorkflowResult.hasWarnings()
- 35. WorkflowResult.debug()
- 36. WorkflowResult.getWorkflowName()
- 37. Exception Hierarchy
- 38. Execution Phases
- 39. Complete Lifecycle
- 40. Quick Reference Tables

**Mejor para**: Referencia técnica, lookups, verificación sintaxis

---

### ✅ GRUPO 12: Debugging y Troubleshooting

#### DOC-14: Debugging y Troubleshooting

**Path**: [14_debugging_troubleshooting_workflow_php-workflow.md](14_debugging_troubleshooting_workflow_php-workflow.md)

**Secciones**:
- 1. Reading Logs
- 2. Log Indicators
- 3. Success Logs
- 4. Warning Logs
- 5. Skip Indicators
- 6. Fail Indicators
- 7. Abort Indicators
- 8. Error: Validation Failed
- 9. Error: Workflow Failed
- 10. Error: Step Skipped
- 11. Error: Data Missing
- 12. Error: Loop Issues
- 13. result->debug()
- 14. ProfileStep Middleware
- 15. Custom Logger Middleware
- 16. Breakpoint Debugging
- 17. State Inspection
- 18. Decision Tree: Step Didn't Execute
- 19. Decision Tree: Workflow Failed
- 20. Decision Tree: Data Disappeared
- 21. Anti-patterns
- 22. Testing Debugging
- 23. Debug Checklist

**Mejor para**: Diagnosis, problem solving, troubleshooting

---

### ✅ GRUPO 13: Advanced Topics

#### DOC-15: Advanced Topics

**Path**: [15_advanced_topics_workflow_php-workflow.md](15_advanced_topics_workflow_php-workflow.md)

**Secciones**:
- 1. State Machines (FSM)
- 2. State Transition Matrix
- 3. Actions by State
- 4. Async Patterns
- 5. Queue-Based Async
- 6. Concurrency Patterns
- 7. Deeply Nested Workflows
- 8. Nesting Best Practices
- 9. Performance Optimization
- 10. Profiling Analysis
- 11. Caching Strategies
- 12. Batch Processing
- 13. Early Exit Patterns
- 14. Memory Management
- 15. Advanced Error Handling
- 16. Circuit Breaker Pattern
- 17. Timeout Handling
- 18. Retry with Backoff
- 19. Security Patterns
- 20. Data Sanitization
- 21. Role-Based Access Control

**Mejor para**: Patrones sofisticados, performance, arquitectura avanzada

---

### ✅ GRUPO 14: Características Opcionales

#### DOC-16: Optional Features

**Path**: [16_optional_features_workflow_php-workflow.md](16_optional_features_workflow_php-workflow.md)

**Secciones**:
- 1. Persistence Overview
- 2. Save & Resume Pattern
- 3. Checkpoint Pattern
- 4. Snapshot & Recovery
- 5. Event Streaming
- 6. Workflow Events Interface
- 7. Event Emitter Pattern
- 8. Pub/Sub Pattern
- 9. Generator Pattern
- 10. Lazy Evaluation
- 11. Pipelined Processing
- 12. GraphQL Integration
- 13. Kafka Integration (Producer)
- 14. Kafka Integration (Consumer)
- 15. Elasticsearch Integration
- 16. DataFrame Processing
- 17. Machine Learning Integration
- 18. Feature Engineering
- 19. ML Pipeline Pattern
- 20. Integration Summary

**Mejor para**: Extensiones, integraciones especializadas, casos avanzados

---

## 🔍 Buscar por Tema

### Por Necesidad

#### ❌ "Mi workflow no funciona"

**Leer en este orden:**
1. [DOC-14, sección 3-12](#doc-14-debugging-y-troubleshooting) - Error messages
2. [DOC-13, sección 37-39](#doc-13-api-reference) - Exception hierarchy
3. [DOC-14, sección 18-20](#doc-14-debugging-y-troubleshooting) - Decision trees

#### 🏗️ "Necesito diseñar un workflow"

**Leer en este orden:**
1. [DOC-01](#doc-01-creación-de-workflows) - Estructura básica
2. [DOC-02](#doc-02-definición-de-steps) - Steps
3. [DOC-10, sección 1-6](#doc-10-best-practices-y-patrones) - Design patterns
4. [DOC-15](#doc-15-advanced-topics) - Advanced patterns

#### 🧪 "Necesito testear"

**Leer en este orden:**
1. [DOC-11](#doc-11-testing-strategies) - Completo
2. [DOC-10, sección 18](#doc-10-best-practices-y-patrones) - Best practices

#### 🚀 "Necesito producción"

**Leer en este orden:**
1. [DOC-12](#doc-12-integración-y-casos-de-uso) - Integration
2. [DOC-10, sección 11-17](#doc-10-best-practices-y-patrones) - Performance & Security
3. [DOC-15, sección 15-21](#doc-15-advanced-topics) - Advanced error handling

#### 🔧 "Necesito extensiones"

**Leer en este orden:**
1. [DOC-06](#doc-06-middleware-y-extensibilidad) - Middleware
2. [DOC-08](#doc-08-dependencias-entre-steps) - Dependencies
3. [DOC-16](#doc-16-optional-features) - Integrations

#### 📊 "Necesito logging/monitoring"

**Leer en este orden:**
1. [DOC-03](#doc-03-logging-y-registros) - Logging system
2. [DOC-14](#doc-14-debugging-y-troubleshooting) - Debugging tools
3. [DOC-10, sección 11-12](#doc-10-best-practices-y-patrones) - Performance monitoring

---

## 📈 Tabla Comparativa de Documentos

| Doc | Grupo | Tipo | Nivel | Palabras | Mejor Para |
|-----|-------|------|-------|----------|-----------|
| 01 | Base | Tutorial | Básico | 9.5k | Inicio |
| 02 | Base | Tutorial | Básico | 11k | Primer step |
| 03 | 1 | Referencia | Intermedio | 6.5k | Logging |
| 04 | 2 | Referencia | Intermedio | 7k | Control flujo |
| 05 | 4 | Tutorial | Intermedio | 8.5k | Nesting |
| 06 | 5 | Referencia | Intermedio | 8k | Middleware |
| 07 | 3 | Referencia | Intermedio | 10k | Loops |
| 08 | 6 | Referencia | Intermedio | 9k | Dependencias |
| 09 | 7 | Referencia | Intermedio | 9k | Results |
| 10 | 8 | Guía | Intermedio | 10k+ | Best practices |
| 11 | 9 | Guía | Intermedio | 9.5k | Testing |
| 12 | 10 | Casos | Avanzado | 8.5k | Real world |
| 13 | 11 | Referencia | Todos | 10k+ | Lookup |
| 14 | 12 | Guía | Todos | 9.5k | Debug |
| 15 | 13 | Tutorial | Avanzado | 8k | Patterns |
| 16 | 14 | Tutorial | Avanzado | 8.5k | Integrations |

---

## 🎓 Rutas de Aprendizaje Sugeridas

### Ruta A: Desarrollador Nuevo (40-60 horas)

1. **Semana 1**: Docs 01-02 (Basics)
2. **Semana 2**: Docs 03-04 (Logging, Control)
3. **Semana 3**: Docs 07, 11 (Loops, Testing)
4. **Semana 4**: Docs 10, 13 (Best Practices, API)
5. **Semana 5**: Docs 12 (Real world)

### Ruta B: Arquitecto Experimentado (20-30 horas)

1. **Día 1**: Docs 01-02, 13 (Basics + API)
2. **Día 2**: Docs 05-06, 08 (Nesting, Middleware, Dependencies)
3. **Día 3**: Docs 10, 15 (Best Practices, Advanced)
4. **Día 4**: Docs 12, 16 (Integration, Optional)

### Ruta C: Troubleshooting (Quick - 5-10 horas)

1. Doc 13 (API Reference) - Lookup problema
2. Doc 14 (Debugging) - Diagnóstico
3. Doc 10 (Best Practices) - Prevención

### Ruta D: Mastery (60-80 horas)

1. Todos en orden (01-16)
2. Implementar ejemplos
3. Tests completos
4. Production deployment

---

## 📞 Cheat Sheets

### Phrases Rápidas

**¿Cómo hago...?**

| Pregunta | Doc | Sección |
|----------|-----|---------|
| ...un workflow? | 01 | 2-4 |
| ...un step? | 02 | 1-3 |
| ...saltar un step? | 04 | 2 |
| ...un loop? | 07 | 2-5 |
| ...workflows anidados? | 05 | 2 |
| ...middleware? | 06 | 4-6 |
| ...validación? | 08 | 1-3 |
| ...testear? | 11 | 2-5 |
| ...en Laravel? | 12 | 3 |
| ...debuggear? | 14 | 1-3 |

### Common Errors

| Error | Doc | Sección |
|-------|-----|---------|
| "Workflow failed" | 14 | 4.2 |
| "Step skipped" | 14 | 4.3 |
| "Data missing" | 14 | 4.5 |
| ValidationException | 14 | 4.1 |
| DependencyException | 08 | 8 |

---

## 📎 Archivos Relacionados en el Workspace

```
/workspaces/atlas/
├── src/                           # Código fuente
│   ├── Workflow.php              # Clase principal
│   ├── WorkflowControl.php       # Control interface
│   ├── Step/WorkflowStep.php     # Step interface
│   ├── State/WorkflowContainer.php
│   ├── State/WorkflowResult.php
│   ├── Exception/                # Jerarquía de excepciones
│   ├── Middleware/               # Middleware built-in
│   └── ...
├── tests/                        # Test suite
│   ├── WorkflowTest.php
│   ├── LoopTest.php
│   ├── NestedWorkflowTest.php
│   └── ...
└── doc_desarrollo/               # DOCUMENTACIÓN COMPLETA
    ├── 00_indice_maestro_documentacion.md  ← ERES AQUÍ
    ├── 00_areas_pendientes_documentacion.md
    ├── 01-02_baseline.md
    ├── 03-10_groups_1-8.md
    ├── 11-12_groups_9-10.md
    ├── 13-16_groups_11-14.md
    └── ...
```

---

## 🏁 Estado Final

| Métrica | Valor |
|---------|-------|
| **Documentos** | 16 ✅ |
| **Grupos** | 14 ✅ |
| **Temas** | 50+ ✅ |
| **Subtemas** | 180+ ✅ |
| **Palabras** | ~110k-130k ✅ |
| **Cobertura** | 100% ✅ |
| **API Completa** | ✅ |
| **Casos Reales** | ✅ |
| **Debugging Guide** | ✅ |
| **Testing Guide** | ✅ |
| **Best Practices** | ✅ |
| **Advanced Topics** | ✅ |
| **Integraciones** | ✅ |

---

## 🎯 Próximos Pasos

### Para el Usuario

1. **Elige tu ruta de aprendizaje** (Ruta A, B, C o D)
2. **Lee los documentos en orden recomendado**
3. **Implementa ejemplos y casos prácticos**
4. **Ejecuta los tests**
5. **Integra en tu proyecto**

### Para Mantenimiento

- Documentos están en `/workspaces/atlas/doc_desarrollo/`
- Usar `00_areas_pendientes_documentacion.md` para tracking
- Usar este `00_indice_maestro_documentacion.md` para navegación
- Todos los documentos tienen secciones numeradas para referencia
- Usar `[DOC-XX, sección Y]` format para cross-referencing

---

**¡La documentación está completa y lista para usar!**

Comienza por [DOC-01: Creación de Workflows](01_creacion_workflow_php-workflow.md) o consulta la [Tabla de Navegación Rápida](#-guía-de-navegación-rápida) para ir directo a lo que necesitas.
