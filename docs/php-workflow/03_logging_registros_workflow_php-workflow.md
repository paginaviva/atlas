# 3. Logging y Registros en php-workflow

## 1. Objetivo

Documentar el sistema de logging y registros que php-workflow proporciona para capturar, organizar y presentar la ejecución completa de workflows, permitiendo auditoría, depuración y visualización de procesos.

---

## 2. Contexto

Una vez que un workflow se crea (Document 01) y sus pasos se definen y ejecutan (Document 02), es necesario un mecanismo robusto para registrar qué ocurrió durante la ejecución. Este sistema debe:

- **Capturar evidencia** de cada paso ejecutado (descripción, estado, razón de fallo)
- **Organizar información** por etapa de workflow (Prepare, Validate, Before, Process, OnSuccess, OnError, After)
- **Agregar contexto** sobre eventos especiales (loops, workflows anidados)
- **Acumular advertencias** que ocurran durante la ejecución
- **Permitir múltiples formatos** de salida (texto legible, gráficos, JSON, etc.)
- **Implementar timing** para medir performance de ejecución

El logging en php-workflow es **en tiempo de ejecución y completamente sistemático**: cada paso ejecutado automáticamente genera un registro, sin intervención del usuario.

---

## 3. Componentes Principales

### 3.1 ExecutionLog - Orquestador Central

Ubicación: [`src/State/ExecutionLog/ExecutionLog.php`](src/State/ExecutionLog/ExecutionLog.php)

**Propósito**: Coordinador central que recibe eventos de ejecución y los organiza en registros persistentes.

**Estructura interna**:

```
ExecutionLog
├── stages[][]          // Step[][] organisados por stage
│   ├── Stage 0 (Prepare)
│   │   └── Step (ejecutado en Prepare)
│   ├── Stage 1 (Validate)
│   │   └── Step (ejecutado en Validate)
│   ├── Stage 7 (Summary)
│   │   └── Step (resumen final)
│   └── ... (8 etapas totales)
├── stepInfo[]          // Contexto pendiente para el próximo Step
├── warnings[][]        // Advertencias agregadas por stage
├── warningsDuringStep  // Contador de warnings para Step actual
└── startAt             // Timestamp para medición de performance
```

**Ciclo de vida**:

1. `__construct(WorkflowState)` - Inicialización con estado
2. `startExecution()` - Inicia timer al comenzar workflow
3. `addStep(stage, Describable, state, reason)` - Registra paso ejecutado
4. `attachStepInfo(info, context)` - Acumula contexto para próximo paso
5. `addWarning(message, workflowReportWarning)` - Registra advertencia
6. `stopExecution()` - Termina timer, agrega información de resumen
7. `debug(OutputFormat)` - Genera salida con formato especificado

### 3.2 Describable - Interfaz de Descripción

Ubicación: [`src/State/ExecutionLog/Describable.php`](src/State/ExecutionLog/Describable.php)

**Propósito**: Contrato para cualquier objeto que puede describirse a sí mismo en un log.

```php
interface Describable
{
    public function getDescription(): string;
}
```

**Implementadores conocidos**:
- `WorkflowStep` (pasos personalizados heredan esta interfaz)
- `Summary` (envases para información agregada)
- `Loop` (ciclos especiales)
- `NestedWorkflow` (workflows anidados)

**Patrón**: Cualquier entidad que puede registrarse en el log **debe implementar** `getDescription()` para proporcionar un nombre humano-legible.

### 3.3 Step - Registro Inmutable de Ejecución

Ubicación: [`src/State/ExecutionLog/Step.php`](src/State/ExecutionLog/Step.php)

**Propósito**: Capsula inmutable que representa un paso ejecutado en el log.

**Estructura**:

```php
class Step implements Describable
{
    private Describable $step;              // El paso ejecutado (WorkflowStep, Summary, etc.)
    private string $state;                  // 'ok' | 'skipped' | 'failed'
    private ?string $reason;                // Razón de fallo/skip (mensaje de excepción)
    private array $stepInfo;                // StepInfo[] contexto (loops, nested wf)
    private int $warnings;                  // Cantidad de warnings durante ejecución
}
```

**Estados posibles**:

- `ExecutionLog::STATE_SUCCESS = 'ok'` - Paso completado exitosamente
- `ExecutionLog::STATE_SKIPPED = 'skipped'` - Paso omitido (SkipStepException)
- `ExecutionLog::STATE_FAILED = 'failed'` - Paso falló (FailStepException o Exception)

**Métodos**:

- `getDescription(): string` - Retorna descripción del paso (delega a $step)
- `getState(): string` - Retorna estado ('ok', 'skipped', 'failed')
- `getReason(): ?string` - Retorna razón de fallo/skip
- `getStepInfo(): array` - Retorna array de StepInfo acumulado
- `getWarnings(): int` - Retorna cantidad de warnings

**Ciclo**: `new Step($step, $state, $reason, $stepInfo, $warningsDuringStep)` - creación automática por ExecutionLog::addStep()

### 3.4 StepInfo - Contexto Especializado

Ubicación: [`src/State/ExecutionLog/StepInfo.php`](src/State/ExecutionLog/StepInfo.php)

**Propósito**: Capsula de información contextual adjunta a un Step para eventos especiales.

**Constantes predefinidas**:

```php
const NESTED_WORKFLOW = 'STEP_NESTED_WORKFLOW';  // Paso es un workflow anidado
const LOOP_START = 'STEP_LOOP_START';            // Inicio de ciclo
const LOOP_ITERATION = 'STEP_LOOP_ITERATION';    // Iteración de ciclo
const LOOP_END = 'STEP_LOOP_END';                // Fin de ciclo
```

**Estructura**:

```php
class StepInfo
{
    private string $info;      // Tipo de info (constante o mensaje)
    private array $context;    // Datos contextuales adicionales
}
```

**Métodos**:

```php
public function __construct(string $info, array $context = []) { }
public function getInfo(): string { return $this->info; }
public function getContext(): array { return $this->context; }
```

**Ejemplos de contexto por tipo**:

- `NESTED_WORKFLOW`: `['result' => WorkflowResult]` - El resultado anidado
- `LOOP_START`: `['description' => string]` - Descripción del loop
- `LOOP_ITERATION`: `['iteration' => int]` - Número de iteración
- `LOOP_END`: `['iterations' => int]` - Total de iteraciones

### 3.5 Summary - Describable para Resúmenes

Ubicación: [`src/State/ExecutionLog/Summary.php`](src/State/ExecutionLog/Summary.php)

**Propósito**: Implementación de Describable que permite registrar información agregada (resúmenes, explicaciones).

**Estructura**:

```php
class Summary implements Describable
{
    private string $description;
}
```

**Uso típico**:

```php
// En Loop.php - registra inicio de ciclo
WorkflowState::getRunningWorkflow()->addExecutionLog(new Summary('Start Loop'));

// En AllowNextExecuteWorkflow - registra ejecución del workflow
WorkflowState::getRunningWorkflow()->addExecutionLog(new Summary('Workflow execution'));

// En Loop.php - registra cada iteración
->addExecutionLog(new Summary("Loop iteration #$iteration"), $state, $reason);
```

---

## 4. Sistema de Advertencias (Warnings)

ExecutionLog mantiene un sistema de dos niveles para advertencias:

### 4.1 Acumulación de Warnings

**Array de warnings** organizado por stage:

```php
private array $warnings = [];  // stage => string[message]
```

**Métodos**:

```php
public function addWarning(string $message, bool $workflowReportWarning = false): void
{
    // Almacena en stage actual
    $this->warnings[$this->workflowState->getStage()][] = $message;
    
    // Si NO es reporte a nivel workflow, incrementa contador del paso
    if (!$workflowReportWarning) {
        $this->warningsDuringStep++;
    }
}
```

**Parámetro `workflowReportWarning`**:

- `false` (defecto): Warning es parte de auditoría del paso (se cuenta en `Step::$warnings`)
- `true`: Warning es reporte a nivel workflow (se agrega al resumen final, no cuenta en paso)

### 4.2 Reporte de Warnings Final

En `stopExecution()`, si hay warnings, se agrega un resumen:

```php
// se construye string como:
"Got {cantidad} warning{s} during the execution:
  Prepare: [warnings en Prepare]
  Validate: [warnings en Validate]
  ...
  After: [warnings en After]"
```

**Acceso**:

- `getWarnings(): array` - Retorna `warnings[][]` completo
- En `WorkflowResult::hasWarnings(): bool`
- En `WorkflowResult::getWarnings(): array`

---

## 5. Sistema de Timing y Performance

ExecutionLog implementa medición automática de tiempo de ejecución:

### 5.1 Captura de Timestamps

```php
private float $startAt;  // Timestamp al iniciar ejecución

public function startExecution(): void
{
    $this->startAt = microtime(true);
}

public function stopExecution(): void
{
    // Calcula diferencia en milisegundos
    $elapsed = 1000 * (microtime(true) - $this->startAt);
    $this->attachStepInfo('Execution time: ' . number_format($elapsed, 5) . 'ms');
}
```

### 5.2 Integración en Workflow

Se invoca automáticamente por `AllowNextExecuteWorkflow` (ejecutor del workflow):

```php
// Al iniciar workflow completo
$workflowState->getExecutionLog()->startExecution();

// Al finalizar (después de etapa After)
$workflowState->getExecutionLog()->stopExecution();
```

**Resultado**: El tiempo total aparece como StepInfo adjunto al resumen final.

---

## 6. Interfaz OutputFormat - Sistema de Formatters

Ubicación: [`src/State/ExecutionLog/OutputFormat/OutputFormat.php`](src/State/ExecutionLog/OutputFormat/OutputFormat.php)

**Propósito**: Abstracción para permitir múltiples formatos de salida del log.

```php
interface OutputFormat
{
    /**
     * @param string $workflowName      // Nombre del workflow
     * @param Step[][] $steps           // Pasos agrupados por stage
     * @return mixed                    // Formato específico (string, array, etc.)
     */
    public function format(string $workflowName, array $steps);
}
```

**Invocación**:

```php
$workflowResult->debug($formatter);
// Equivalente a:
$workflowState->getExecutionLog()->debug($formatter);
```

**Flujo**:

```
WorkflowResult::debug(OutputFormat?)
  ├─ Si formatter = null: usa StringLog por defecto
  └─ executionLog.debug(formatter)
      └─ formatter.format(workflowName, stages[][])
```

---

## 7. Formatter 1: StringLog - Salida Legible

Ubicación: [`src/State/ExecutionLog/OutputFormat/StringLog.php`](src/State/ExecutionLog/OutputFormat/StringLog.php)

**Propósito**: Formatea log en texto legible con indentación jerárquica.

### 7.1 Comportamiento

**Salida por defecto** cuando no se especifica formatter:

```
Process log for workflow 'MyWorkflow':

Prepare:
  - UserValidationStep: ok (1 warning)
  - ConfigurationStep: ok

Validate:
  - DataValidationStep: ok

Process:
  - MainProcessStep: ok
      - Loop finished after 5 iterations

After:
  - CleanupStep: ok
      - Execution time: 1234.56789ms
```

### 7.2 Método principal

```php
public function format(string $workflowName, array $steps): string
{
    // Itera cada etapa (Prepare, Validate, Before, Process, OnSuccess, OnError, After, Summary)
    // Para cada etapa, itera steps
    // Para cada step, formatea con estado y razón (si aplica)
    // Adjunta StepInfo interpretado (loops, nested workflows)
    // Retorna string final
}
```

### 7.3 Lógica de formatInfo

Interpreta tipos especiales de StepInfo:

| Info | Acción |
|------|--------|
| `NESTED_WORKFLOW` | Recibe resultado anidado, aplica offset de indentación |
| `LOOP_START` | Incrementa indentación para iteraciones |
| `LOOP_ITERATION` | Retorna null (no imprime) |
| `LOOP_END` | Decrementa indentación, reporta # iteraciones |
| Otra (string) | Imprime como `    - {info}` |

**Indentación**: Se mantiene estado en `$indentation` para anidar loops/workflows.

---

## 8. Formatter 2: GraphViz - Visualización de Grafos

Ubicación: [`src/State/ExecutionLog/OutputFormat/GraphViz.php`](src/State/ExecutionLog/OutputFormat/GraphViz.php)

**Propósito**: Genera script DOT (GraphViz) que representa la ejecución como grafo dirigido.

### 8.1 Salida

Genera DOT script que puede visualizarse con `dot` command:

```
digraph "MyWorkflow" {
  0 [label="MyWorkflow"]
  subgraph cluster_0 {
    label = "Prepare"
    1 [label="UserValidationStep" shape="box" color="#FFFF00"]
    ...
  }
  subgraph cluster_1 {
    label = "Validate"
    ...
  }
  subgraph cluster_loop_2 {
    label = "Loop"
    ...
  }
  0 -> 1
  1 -> 2
  ...
}
```

### 8.2 Manejo de Loops

- `LOOP_START`: Abre `subgraph cluster_loop_N { label = "Loop" }`
- `LOOP_ITERATION`: Agrega link de feedback (próximo step → iteración inicial)
- `LOOP_END`: Cierra subgraph

### 8.3 Manejo de Workflows Anidados

- `NESTED_WORKFLOW`: Embebe el grafo anidado dentro de un `subgraph cluster`
- Mantiene estructura de clusters de etapas internas

### 8.4 Colores de Estados

```php
sprintf('    %s [label=%s shape="box" color="%s"]',
    $stepIndex,
    $step->getDescription(),
    // Color basado en estado...
)
```

Mapeo (inferido del código):
- Estado 'ok': Color específico (probablemente verde #00FF00)
- Estado 'skipped': Otro color (#FFFF00 amarillo probable)
- Estado 'failed': Otro color (#FF0000 rojo probable)

---

## 9. Formatter 3: WorkflowGraph - Exportación a SVG

Ubicación: [`src/State/ExecutionLog/OutputFormat/WorkflowGraph.php`](src/State/ExecutionLog/OutputFormat/WorkflowGraph.php)

**Propósito**: Convierte DOT script (GraphViz) a SVG e invoca GraphViz CLI.

### 9.1 Flujo de Ejecución

```
WorkflowGraph::format()
  ├─ Crea GraphViz formatter
  ├─ Obtiene script DOT
  ├─ Escribe a archivo temporal
  ├─ Invoca: dot -T svg temp_file -o output.svg
  ├─ Elimina temporal
  └─ Retorna ruta a SVG generado
```

### 9.2 Requisitos del Sistema

Requiere herramienta `dot` disponible en PATH:

```bash
system('dot -T svg ' . escapeshellarg($tmp) . ' -o ' . escapeshellarg($file), $ret);
```

**Si `dot` no existe**: Lanza `Exception('Unable to invoke "dot"...')`

### 9.3 Ubicación del Output

```php
public function __construct(string $path)
{
    $this->path = $path;  // Directorio base
}

// Genera: {path}/workflow_name_UNIQUE_ID.svg
```

---

## 10. Integración: Flujo Completo del Logging

### 10.1 Captura de Pasos Ejecutados

En `StepExecutionTrait::wrapStepExecution()`:

```
wrapStepExecution(WorkflowStep, WorkflowState)
  ├─ Ejecuta step con middleware
  ├─ Si SkipStepException: addExecutionLog(..., STATE_SKIPPED, $msg)
  ├─ Si FailStepException: addExecutionLog(..., STATE_FAILED, $msg)
  ├─ Si Exception: addExecutionLog(..., STATE_FAILED, $msg)
  └─ Si éxito: addExecutionLog($step)  [STATE_SUCCESS implícito]
```

### 10.2 Mapeo WorkflowState.addExecutionLog() → ExecutionLog.addStep()

En `WorkflowState`:

```php
public function addExecutionLog(
    Describable $step,
    string $state = ExecutionLog::STATE_SUCCESS,
    ?string $reason = null
): void {
    $this->executionLog->addStep($this->stage, $step, $state, $reason);
}
```

**Parámetros automáticos**:
- `stage`: Stage actual de WorkflowState
- `stepInfo`: Array acumulado de StepInfo pendientes
- `warningsDuringStep`: Contador de warnings del paso

### 10.3 Ejemplo: Ejecución Completa

```
Workflow comienza
  ├─ startExecution() → inicia timer
  ├─ STAGE_PREPARE
  │   ├─ Ejecuta Step A
  │   │   └─ wrapStepExecution(A) → addExecutionLog(A, ok)
  │   │       ├─ Step A agregado a stages[PREPARE]
  │   │       └─ stepInfo[] limpiado
  │   └─ Ejecuta Step B
  │       └─ addExecutionLog(B, skipped, "User cancelled")
  ├─ STAGE_VALIDATE
  │   └─ Ejecuta Step C
  └─ STAGE_AFTER
      ├─ Ejecuta Step D
      └─ stopExecution()
          ├─ Agrega timing info
          ├─ Agrega resumen de warnings
          └─ Complete log está en stages[][]

Salida/Debug:
  └─ result.debug() → StringLog por defecto
      ├─ Itera stages[0..7]
      └─ Retorna string legible
```

---

## 11. Métodos WorkflowControl para Logging

Ubicación: [`src/WorkflowControl.php`](src/WorkflowControl.php)

**Interfaz pública** para que steps accedan a logging:

```php
class WorkflowControl
{
    // Acceso a logging
    public function attachStepInfo(string $info, array $context = []): void
    {
        $this->workflowState->getExecutionLog()->attachStepInfo($info, $context);
    }

    public function addWarning(string $message): void
    {
        $this->workflowState->getExecutionLog()->addWarning($message);
    }
}
```

**Uso en steps personalizados**:

```php
class MyStep implements WorkflowStep
{
    public function run(WorkflowControl $control, WorkflowContainer $container): void
    {
        // Adjunta contexto
        $control->attachStepInfo('Processing order #123');
        
        // Reporta advertencia
        if ($problematic) {
            $control->addWarning('Potential data consistency issue detected');
        }
    }
}
```

---

## 12. Acceso al Log desde WorkflowResult

Ubicación: [`src/State/WorkflowResult.php`](src/State/WorkflowResult.php)

**Métodos de acceso**:

```php
class WorkflowResult
{
    // Genera salida del log
    public function debug(OutputFormat $formatter = null): mixed
    {
        return $this->workflowState->getExecutionLog()->debug(
            $formatter ?? new StringLog()
        );
    }

    // Verifica si hay warnings
    public function hasWarnings(): bool
    {
        return count($this->workflowState->getExecutionLog()->getWarnings()) > 0;
    }

    // Accede a warnings completos
    public function getWarnings(): array
    {
        return $this->workflowState->getExecutionLog()->getWarnings();
    }
}
```

---

## 13. Casos Especiales: Loops

Ubicación: [`src/Step/Loop.php`](src/Step/Loop.php)

### 13.1 Registro de Loops

```
Loop ejecutándose
  ├─ attachStepInfo(LOOP_START, ['description' => ...])
  ├─ ITERACIÓN 1
  │   ├─ Ejecuta inner step X
  │   ├─ attachStepInfo(LOOP_ITERATION, ['iteration' => 1])
  │   └─ addExecutionLog(Summary("Loop iteration #1"), state)
  ├─ ITERACIÓN 2
  │   └─ ... (igual)
  ├─ ...
  └─ attachStepInfo(LOOP_END, ['iterations' => N])
```

### 13.2 Impacto en StringLog

- `LOOP_START`: Aumenta indentación en 2 espacios
- Iteraciones se imprimen indentadas
- `LOOP_END`: Disminuye indentación, reporta "Loop finished after N iteration{s}"

### 13.3 Impacto en GraphViz

- `LOOP_START`: Abre `subgraph cluster_loop_N`
- `LOOP_ITERATION`: Agrega link de feedback (loop backlink)
- `LOOP_END`: Cierra subgraph

---

## 14. Casos Especiales: Workflows Anidados

Ubicación: [`src/Step/NestedWorkflow.php`](src/Step/NestedWorkflow.php)

### 14.1 Registro de Workflows Anidados

```
NestedWorkflow ejecutándose
  ├─ Ejecuta workflow anidado COMPLETO
  │   └─ Genera su propio log (stages[][], warnings)
  └─ attachStepInfo(NESTED_WORKFLOW, ['result' => WorkflowResult])
```

### 14.2 Acceso al Log Anidado

```php
// En StringLog::formatInfo()
case StepInfo::NESTED_WORKFLOW:
    $nestedResult = $info->getContext()['result'];
    // Recursivamente formatea el log anidado con offset
    return nestedResult->debug($this);  // Mismo formatter
```

**Resultado**: Anidamiento visual completo en StringLog.

---

## 15. Hallazgos Clave

### 15.1 Captura Automática

- **Cada step** registrado automáticamente (sin código usuario)
- **Cada excepción** capturada y registrada con razón
- **Cada warning** acumulado automáticamente
- **Timing automático** de ejecución completa

### 15.2 Estadística por Estado

ExecutionLog mantiene contador exacto de:
- Pasos exitosos (STATE_SUCCESS)
- Pasos omitidos (STATE_SKIPPED)
- Pasos fallidos (STATE_FAILED)
- Warnings por stage

### 15.3 Estructura por Etapas

El log está **siempre organizado por etapas** (nunca plano):

```
stages[0] = [Steps en PREPARE]
stages[1] = [Steps en VALIDATE]
stages[2] = [Steps en BEFORE]
stages[3] = [Steps en PROCESS]
stages[4] = [Steps en ON_ERROR o ON_SUCCESS]
stages[5] = [Steps en otro ON_ERROR o ON_SUCCESS]
stages[6] = [Steps en ON_ERROR]
stages[7] = [Steps en AFTER]
stages[8] = [Summary entries]
```

### 15.4 Contexto Acumulativo

`stepInfo[]` actúa como **buffer acumulativo** entre pasos:

- Se adjunta contexto en paso actual
- Se adjunta al Step al registrar
- **Se limpia automáticamente** después
- Permite múltiples adjuntos antes de registro

### 15.5 Warnings Separados

Dos niveles de warnings:

1. **Step-level** (`warningsDuringStep`): Auditoría del paso en ejecución
2. **Workflow-level** (`workflowReportWarning=true`): Reporte final agregado

Permite auditoría granular sin inflar cuota de warnings paso a paso.

### 15.6 Extensibilidad de Formatters

OutputFormat es simple (1 método) → fácil crear formatos personalizados:

```php
class JSONFormatter implements OutputFormat {
    public function format(string $workflowName, array $steps) {
        return json_encode([...]);
    }
}

$result->debug(new JSONFormatter());
```

### 15.7 Timing Precisión

Usa `microtime(true)` (microsegundos) → precisión en milisegundos:

```php
$elapsed = 1000 * (microtime(true) - $startAt);  // Convierte a ms
```

### 15.8 Stateless y Reusable

ExecutionLog instance es **única por ejecución**, pero:

- Formatter puede reutilizarse (stateless)
- OutputFormat interface permite extensión
- El mismo workflow genera diferentes logs en diferentes ejecuciones

---

## 16. Dependencies y Archivos Relacionados

### 16.1 Dependencias Internas

```
ExecutionLog
├─ Step (crea internally)
├─ StepInfo (acumula)
├─ OutputFormat (interface)
├─ WorkflowState (recibe en constructor)
└─ Describable (interface recibida)

Formatters (OutputFormat impl)
├─ StringLog
│   ├─ StepInfo (interpreta constantes)
│   └─ WorkflowResult (accede para nested)
├─ GraphViz
│   ├─ StepInfo (mapea a DOT clusters)
│   └─ WorkflowResult (embebe grafo anidado)
└─ WorkflowGraph
    ├─ GraphViz (genera script)
    └─ system('dot') (requiere programa externo)
```

### 16.2 Archivos Clave

| Archivo | Propósito |
|---------|-----------|
| `ExecutionLog.php` | Orquestador central |
| `Describable.php` | Interfaz de descripción |
| `Step.php` | Registro de paso inmutable |
| `StepInfo.php` | Contexto especializado |
| `Summary.php` | Resumen como Describable |
| `OutputFormat.php` | Interfaz de formatter |
| `StringLog.php` | Formatter texto |
| `GraphViz.php` | Formatter DOT script |
| `WorkflowGraph.php` | Formatter SVG (requiere dot CLI) |

### 16.3 Puntos de Integración

| Componente | Integración |
|-----------|-------------|
| `WorkflowState` | Contiene ExecutionLog, método `addExecutionLog()` |
| `StepExecutionTrait` | Llama a `addExecutionLog()` en cada resultado |
| `WorkflowResult` | Accede a `debug()` para salida |
| `WorkflowControl` | Expone `attachStepInfo()` y `addWarning()` a steps |
| `Loop` | Adjunta LOOP_START/LOOP_ITERATION/LOOP_END |
| `NestedWorkflow` | Adjunta NESTED_WORKFLOW con WorkflowResult |
| `ProfileStep` | Middleware que adjunta timing de paso |

---

## 17. Flujo Temporal de Ejecución

```
┌─ INICIO DE WORKFLOW ──────────────────────────────────────┐
│ ExecutionLog::startExecution()  → Inicia microtime timer   │
├──────────────────────────────────────────────────────────┤
│ STAGE_PREPARE:                                           │
│   Step A:  wrapStepExecution(A)                          │
│     ├─ Middleware chain                                 │
│     └─ addExecutionLog(A, ok) → new Step agregado       │
│   Step B:  wrapStepExecution(B)                          │
│     ├─ Excepción SkipStep                               │
│     └─ addExecutionLog(B, skipped, msg)                 │
├──────────────────────────────────────────────────────────┤
│ STAGE_VALIDATE: [...steps...]                           │
├──────────────────────────────────────────────────────────┤
│ STAGE_PROCESS: [Step con Loop]                          │
│   Loop Step:  wrapStepExecution(Loop)                   │
│     ├─ attachStepInfo(LOOP_START)                       │
│     ├─ Iteración 1:                                     │
│     │   ├─ Inner steps                                  │
│     │   └─ attachStepInfo(LOOP_ITERATION)               │
│     ├─ Iteración 2: [igual]                             │
│     └─ attachStepInfo(LOOP_END)                         │
│   addExecutionLog(Loop, ok) → Step con todo adjunto      │
├──────────────────────────────────────────────────────────┤
│ STAGE_AFTER:  [...steps...]                             │
├──────────────────────────────────────────────────────────┤
│ ExecutionLog::stopExecution()                            │
│   ├─ Calcula elapsed time en ms                         │
│   ├─ Adjunta 'Execution time: X.XXXXXms'               │
│   ├─ Si hay warnings, agrega resumen                    │
│   └─ Complete log en stages[][]                         │
└──────────────────────────────────────────────────────────┘
                          ↓
                  WorkflowResult::
                  debug(FormatterOpt?)
                      ├─ StringLog (default)
                      ├─ GraphViz
                      ├─ WorkflowGraph (SVG)
                      └─ Custom OutputFormat
```

---

## 18. Conclusiones

El sistema de logging de php-workflow es un **mecanismo robusto, automático y extensible** que captura la ejecución completa de workflows:

### 18.1 Fortalezas

- **Captura automática**: Cada paso, cada excepción, cada warning registrado sin intervención
- **Estructura ordenada**: Organización clara por etapas facilita auditoría
- **Extensible**: Interface OutputFormat permite formatters personalizados
- **Performance integrada**: Timing automático sin código usuario
- **Contexto enriquecido**: StepInfo permite adjuntar información especializada
- **Múltiples formatos**: Texto legible (StringLog), visualización (GraphViz/SVG), extensible

### 18.2 Patrones Observados

- **Patrón Describable**: Contrato simple para objetos que pueden describirse
- **Patrón Buffer**: `stepInfo[]` como buffer acumulativo entre registros
- **Patrón Acumulador**: Warnings organizados por etapa para reporting final
- **Patrón Formatter**: OutputFormat abstracción de formato especificación
- **Patrón Anidamiento**: Logs anidados soportan workflows anidados recursivamente

### 18.3 Casos de Uso

1. **Auditoría**: Aceso a `getWarnings()` por revisión
2. **Debugging**: `debug(StringLog)` visualiza todo lo que ocurrió
3. **Monitoring**: Timing automático permite tracking de performance
4. **Visualización**: GraphViz genera gráficos de ejecución
5. **Reporting**: Acceso público a log favorece generación de reportes

### 18.4 Integración Necesaria para Uso

```php
$workflow = new Workflow();
// ... definir workflow ...

$result = $workflow->run();

// Mostrar log en texto
echo $result->debug();

// O en SVG (requiere 'dot' CLI)
$result->debug(new WorkflowGraph('/path/to/output'));

// O verificar warnings
if ($result->hasWarnings()) {
    print_r($result->getWarnings());
}
```

### 18.5 Consideraciones de Diseño

- **Inmutabilidad**: Step es inmutable una vez creado (copia de stepInfo y warnings)
- **Timing de precisión**: microtime(true) para submilisegundo accuracy
- **CLI dependency**: WorkflowGraph requiere `dot` en PATH (GraphViz externo)
- **Memory**: logs acumulan en memoria (stages[][], warnings[]) - considerar para workflows largos
- **Stateless formatters**: Formatters reutilizables entre ejecuciones

---

## 19. Referencias de Código

### Ubicaciones Clave

- Core: `src/State/ExecutionLog/ExecutionLog.php`
- Interfaces: `src/State/ExecutionLog/Describable.php`
- Registros: `src/State/ExecutionLog/Step.php`, `StepInfo.php`, `Summary.php`
- Formatters: `src/State/ExecutionLog/OutputFormat/*.php`
- Integración: `src/WorkflowControl.php`, `src/Step/StepExecutionTrait.php`
- Acceso: `src/State/WorkflowResult.php`

### Tests Recomendados

Para comprender comportamiento en profundidad:
- `tests/WorkflowTest.php` - Muestra logs completos
- `tests/LoopTest.php` - Muestra registro de loops
- `tests/NestedWorkflowTest.php` - Muestra anidamiento

