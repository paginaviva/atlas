# 5. Workflows Anidados en php-workflow

## 1. Objetivo

Documentar el mecanismo completo de workflows anidados que permite incrustar workflows dentro de workflows, creando estructuras jerárquicas sophisticadas con herencia de contexto, propagación de estado y manejo automático de errores.

---

## 2. Contexto

Un workflow puede contener pasos complejos. php-workflow permite que un paso sea un **workflow completo** indepencientemente funcional pero incrustado dentro de otro. Esto habilita:

- **Reutilización**: Workflows existentes pueden componerse sin reescribir
- **Encapsulación**: Lógica compleja encapsulada en un step
- **Cascada**: Múltiples niveles de anidamiento (A contiene B contiene C)
- **Herencia de contexto**: El workflow anidado accede al contenedor del padre
- **Propagación automática**: Estados, errores y logs se propagan hacia arriba
- **Separación de responsabilidades**: Cada workflow puede tener su propia estructura

El concepto es **composición de workflows**: en lugar de pasos simples, usas workflows como pasos.

---

## 3. NestedWorkflow - Componente Principal

Ubicación: [`src/Step/NestedWorkflow.php`](src/Step/NestedWorkflow.php)

**Propósito**: Implementación de WorkflowStep que ejecuta otro workflow manteniendo contexto.

### 3.1 Constructor y Parámetros

```php
class NestedWorkflow implements WorkflowStep
{
    private ExecutableWorkflow $nestedWorkflow;
    private ?WorkflowContainer $container;

    /**
     * @param ExecutableWorkflow $nestedWorkflow - El workflow a ejecutar
     * @param WorkflowContainer|null $container - Opcional: contenedor con datos específicos
     */
    public function __construct(ExecutableWorkflow $nestedWorkflow, ?WorkflowContainer $container = null)
    {
        $this->nestedWorkflow = $nestedWorkflow;
        $this->container = $container;
    }
}
```

**Parámetros**:

- `$nestedWorkflow`: Workflow completo (objeto que implementa ExecutableWorkflow)
- `$container`: Contenedor opcional específico para el workflow anidado

### 3.2 Ejecución

```php
public function run(WorkflowControl $control, WorkflowContainer $container): void
{
    try {
        // Crea NestedContainer que hereda del padre
        $this->workflowResult = $this->nestedWorkflow->executeWorkflow(
            new NestedContainer($container, $this->container),
            $container->get('__internalExecutionConfiguration')['throwOnFailure'],
        );
    } catch (WorkflowException $exception) {
        // Captura excepción del workflow anidado
        $this->workflowResult = $exception->getWorkflowResult();
    }

    // Adjunta resultado como contexto en logs
    $control->attachStepInfo(StepInfo::NESTED_WORKFLOW, ['result' => $this->workflowResult]);

    // Reporta warnings del workflow anidado
    if ($this->workflowResult->getWarnings()) {
        $warnings = count($this->workflowResult->getWarnings(), COUNT_RECURSIVE) -
            count($this->workflowResult->getWarnings());

        $control->warning(
            sprintf(
                "Nested workflow '%s' emitted %s warning%s",
                $this->workflowResult->getWorkflowName(),
                $warnings,
                $warnings > 1 ? 's' : '',
            ),
        );
    }

    // Si workflow anidado falló, marca este step como fallido
    if (!$this->workflowResult->success()) {
        $control->failStep("Nested workflow '{$this->workflowResult->getWorkflowName()}' failed");
    }
}
```

**Lógica principal**:

1. **Crea NestedContainer** que hereda del contenedor padre
2. **Ejecuta workflow anidado** con ese contenedor
3. **Captura excepciones** de ejecución del workflow
4. **Adjunta resultado** para logging
5. **Reporta warnings** como warning de step
6. **Propaga fallo** si el workflow anidado falló

### 3.3 Acceso al Resultado

```php
public function getNestedWorkflowResult(): WorkflowResult
{
    return $this->workflowResult;
}
```

Permite acceso a `WorkflowResult` completo del workflow anidado:

```php
$result = $workflow->executeWorkflow();
$lastStep = $result->getLastStep();  // Instancia de NestedWorkflow

if ($lastStep instanceof NestedWorkflow) {
    $nestedResult = $lastStep->getNestedWorkflowResult();
    echo "Nested workflow success: " . ($nestedResult->success() ? 'yes' : 'no');
    print_r($nestedResult->getWarnings());
}
```

---

## 4. NestedContainer - Contenedor Heredado

Ubicación: [`src/State/NestedContainer.php`](src/State/NestedContainer.php)

**Propósito**: Contenedor que implementa herencia de contexto del workflow padre.

### 4.1 Estructura Interna

```php
class NestedContainer extends WorkflowContainer
{
    private WorkflowContainer $parentContainer;
    private ?WorkflowContainer $container;

    public function __construct(WorkflowContainer $parentContainer, ?WorkflowContainer $container)
    {
        $this->parentContainer = $parentContainer;  // Contexto del padre
        $this->container = $container;              // Contexto específico (opcional)
    }
}
```

**Dos contenedores**:

1. **$parentContainer**: Contexto del workflow padre (siempre presente)
2. **$container**: Contenedor específico del workflow anidado (puede ser null)

### 4.2 Comportamiento de get()

Lookup en cascada:

```php
public function get(string $key)
{
    if (!$this->container) {
        // Si no hay contenedor específico, busca en padre
        return $this->parentContainer->get($key);
    }

    // Si hay contenedor específico, intenta ahí primero
    return $this->container->get($key) ?? $this->parentContainer->get($key);
}
```

**Orden de búsqueda**:

```
Solicitud: get($key)
  ├─ Si $container existe:
  │   ├─ Busca en $container
  │   └─ Si no encuentra, busca en $parentContainer
  └─ Si $container = null:
      └─ Busca en $parentContainer
```

**Ejemplo**:

```php
// Setup
$parent = (new WorkflowContainer())
    ->set('shared-key', 'from-parent')
    ->set('color', 'blue');

$nested = (new WorkflowContainer())
    ->set('color', 'red')
    ->set('nested-key', 'only-in-nested');

$nested_container = new NestedContainer($parent, $nested);

// Búsquedas
$nested_container->get('shared-key');   // 'from-parent' (solo en parent)
$nested_container->get('color');        // 'red' (en nested, sobreescribe parent)
$nested_container->get('nested-key');   // 'only-in-nested' (solo en nested)
$nested_container->get('unknown');      // null (en ninguno)
```

### 4.3 Comportamiento de set()

Doble escritura:

```php
public function set(string $key, $value): WorkflowContainer
{
    if ($this->container) {
        $this->container->set($key, $value);  // Escribe en contenedor específico
    }

    $this->parentContainer->set($key, $value);  // SIEMPRE escribe en padre

    return $this;
}
```

**Comportamiento**:

- **Con contenedor específico**: Escribe en AMBOS (nested Y parent)
- **Sin contenedor específico**: Escribe solo en parent

**Ejemplo**:

```php
$parent = new WorkflowContainer();
$nested = new WorkflowContainer();
$nested_container = new NestedContainer($parent, $nested);

$nested_container->set('key', 'value');

// Resultado:
$nested->get('key');    // 'value' (escrito)
$parent->get('key');    // 'value' (escrito)
```

**Implicación**: Los datos creados en el workflow anidado se **propagan automáticamente** al padre.

### 4.4 Métodos Personalizados - __call()

```php
public function __call(string $name, array $arguments)
{
    if ($this->container && method_exists($this->container, $name)) {
        return $this->container->{$name}(...$arguments);
    }

    return $this->parentContainer->{$name}(...$arguments);
}
```

**Propósito**: Delegar métodos personalizados a los contenedores subyacentes.

**Comportamiento**:

```
Llamada: $container->customMethod(...args)
  ├─ Si $container existe Y tiene el método:
  │   └─ Ejecuta $container->customMethod()
  └─ Sino:
      └─ Ejecuta $parentContainer->customMethod()
```

**Uso típico**:

```php
class CustomContainer extends WorkflowContainer {
    public function getParentData(): string {
        return 'parent-data';
    }
}

class NestedSpecificContainer extends WorkflowContainer {
    public function getNestedData(): string {
        return 'nested-data';
    }
}

$parent = new CustomContainer();
$nested = new NestedSpecificContainer();
$nested_container = new NestedContainer($parent, $nested);

// Métodos personalizados disponibles:
$nested_container->getParentData();   // 'parent-data' (del parent vía fallback)
$nested_container->getNestedData();   // 'nested-data' (del nested específico)
```

---

## 5. Creación de Workflows Anidados

### 5.1 Patrón Básico

```php
$inner_workflow = new Workflow('inner-process');
$inner_workflow
    ->prepare(...)
    ->process(...)
    ->after(...);

$outer_workflow = new Workflow('outer-process');
$outer_workflow
    ->prepare(...)
    ->process(
        new NestedWorkflow($inner_workflow),
    )
    ->after(...);

// Ejecución
$result = $outer_workflow->executeWorkflow();
```

### 5.2 Con Contenedor Específico

```php
$nested_container = new class() extends WorkflowContainer {
    public function getSpecificData(): string {
        return 'specific-config';
    }
};

$outer_workflow
    ->process(
        new NestedWorkflow($inner_workflow, $nested_container),
    );
```

### 5.3 Sin Contenedor Específico

```php
// $container = null → hereda completamente del padre
new NestedWorkflow($inner_workflow)  // null implícito
```

---

## 6. Propagación de Datos: Herencia vs Actualización

### 6.1 Lectura: Inheritancia

El workflow anidado puede leer datos del padre:

```php
$parent_container = (new WorkflowContainer())
    ->set('request-id', 'REQ-123')
    ->set('user-id', 42);

$inner_workflow = new Workflow('nested');
$inner_workflow->process($custom_step);

$outer_workflow = new Workflow('outer');
$outer_workflow->process(
    new NestedWorkflow($inner_workflow),
)->executeWorkflow($parent_container);

// En $custom_step del workflow anidado:
// $container->get('request-id')  // 'REQ-123' (heredado del padre)
// $container->get('user-id')     // 42 (heredado del padre)
```

### 6.2 Escritura: Actualización del Padre

El workflow anidado cuando escribe, actualiza el padre:

```php
$inner_workflow = new Workflow('nested');
$inner_workflow->process(
    new class implements WorkflowStep {
        public function run($control, $container) {
            $container->set('processed-data', 'result-123');
        }
    }
);

$outer_workflow = new Workflow('outer');
$outer_workflow
    ->process(
        new NestedWorkflow($inner_workflow),
    )
    ->process($verify_step)  // Verifica resultado
    ->executeWorkflow();

// En $verify_step (en el workflow padre):
// $container->get('processed-data')  // 'result-123' (actualizado por nested)
```

### 6.3 Tabla: Herencia vs Actualización

| Operación | Contenedor Específico | Sin Específico | Resultado |
|-----------|----------------------|----------------|-----------|
| `get('key')` en nested | Intenta nested, luego parent | Del parent | Acceso a ambos |
| `set('key', val)` en nested | Escribe en nested Y parent | Escribe en parent | Propagación automática |
| `get('key')` en padre después | El padre tiene el valor | El padre tiene el valor | Sincronización garantizada |

---

## 7. Cascada de Workflows (Anidamiento Múltiple)

Workflows pueden anidarse recursivamente en múltiples niveles:

```
Workflow A
  ├─ Step 1
  ├─ NestedWorkflow(B)
  │   ├─ Workflow B
  │   ├─ NestedWorkflow(C)
  │   │   ├─ Workflow C
  │   │   └─ Process(...)
  │   └─ After(...)
  └─ Step 2
```

### 7.1 Containers en Cascada

```php
// A contiene B, B contiene C
$container_c = new NestedContainer(container_b_instance, null);
$container_b = new NestedContainer(container_a, null);

// Lookup en C:
$container_c->get('key')
  ├─ Busca en container_b
  │   ├─ Busca en container_a
  │   └─ Si no, retorna null
  └─ ...
```

**Implicación**: Acceso transitivo a todos los ancestros.

### 7.2 Ejemplo Práctico: Approval Chain

```php
$final_approval = new Workflow('final-approval');
$final_approval->process(new ApproveStep());

$director_review = new Workflow('director-review');
$director_review->process(
    new ReviewStep(),
    new NestedWorkflow($final_approval),  // A contiene B
);

$manager_review = new Workflow('manager-review');
$manager_review->process(
    new ReviewStep(),
    new NestedWorkflow($director_review),  // B contiene C
);

// Ejecución
$result = $manager_review->executeWorkflow($data);
```

**Flujo**:

```
manager_review
  ├─ ReviewStep (manager nivel)
  └─ NestedWorkflow(director_review)
      ├─ ReviewStep (director nivel)
      └─ NestedWorkflow(final_approval)
          └─ ApproveStep (final)
```

---

## 8. Container Hermanos (Paralelo, no Anidado)

Diferentes a workflows anidados, es decir, workflows hermanos que comparten datos:

```php
$container = (new WorkflowContainer())
    ->set('shared-data', 'value-1');

// Workflow 1
$workflow_1 = new Workflow('process-1');
$workflow_1->process(...)->executeWorkflow($container);

// Workflow 2 (usa contenedor actualizado por workflow 1)
$workflow_2 = new Workflow('process-2');
$workflow_2->process(...)->executeWorkflow($container);

// $container fue actualizado por workflow_1
// workflow_2 ve cambios de workflow_1
```

**Diferencia de anidados**:

- **Anidados**: Workflow B se ejecuta DENTRO de Step de Workflow A
- **Hermanos**: Workflow A y B se ejecutan secuencialmente en código usuario

---

## 9. Propagación de Errores en Workflows Anidados

### 9.1 Captura Automática de Excepciones

```php
// En NestedWorkflow::run()
try {
    $this->workflowResult = $this->nestedWorkflow->executeWorkflow(...);
} catch (WorkflowException $exception) {
    // Captura la excepción automáticamente
    $this->workflowResult = $exception->getWorkflowResult();
}
```

**Comportamiento**:

- Si el workflow anidado falla → se captura la `WorkflowException`
- El resultado se almacena en `$workflowResult`
- El step NO lanza excepción, pero marca como fallido

### 9.2 Lógica de Propagación Condicional

```php
if (!$this->workflowResult->success()) {
    $control->failStep("Nested workflow '{$this->workflowResult->getWorkflowName()}' failed");
}
```

**Resultado**:

- Si workflow anidado falló ✗ → `failStep()` → el NestedWorkflow step falla
- Si el NestedWorkflow step falla → el workflows padre maneja según su etapa

**Tabla: Propagación por Etapa del Padre**

| Etapa Padre | NestedWorkflow Falla | Acción |
|-------------|---------------------|--------|
| Prepare | ✗ | failStep propaga → workflow aborta |
| Validate | ✗ | failStep propaga → workflow aborta |
| Before | ✗ | failStep propaga → workflow aborta |
| Process | ✗ | failStep capturada → setProcessException → OnError se ejecuta |
| OnSuccess/OnError/After | ✗ | failStep capturada → warning agregado |

---

## 10. Logging y Visualización de Workflows Anidados

### 10.1 StepInfo.NESTED_WORKFLOW

En logs, los workflows anidados se marcan especialmente:

```php
$control->attachStepInfo(
    StepInfo::NESTED_WORKFLOW,
    ['result' => $this->workflowResult]
);
```

### 10.2 Salida en StringLog

```
Process log for workflow 'outer':

Before:
  - Execute nested workflow: ok
    - Process log for workflow 'inner':
      Before:
        - inner-step: ok
      Process:
        - inner-process: ok
      
      Summary:
        - Workflow execution: ok
          - Execution time: 123.45ms

After:
  - outer-after-step: ok
```

**Características**:

- El log del workflow anidado se anida con indentación
- Se muestra el nombre exacto del workflow
- Se incluye el resumen del workflow anidado
- Warnings del anidado se reportan

### 10.3 Salida en GraphViz

Los workflows anidados se representan como subgrafos:

```
digraph "outer" {
  subgraph cluster_0 {
    label = "Before"
    1 [label="Execute nested workflow"]
    
    subgraph cluster_nested {
      label = "Nested workflow"
      // Nodos del workflow anidado aquí
      2 [label="inner-step"]
      3 [label="inner-process"]
    }
  }
}
```

---

## 11. Warnings en Workflows Anidados

### 11.1 Agregación de Warnings

```php
if ($this->workflowResult->getWarnings()) {
    $warnings = count($this->workflowResult->getWarnings(), COUNT_RECURSIVE) -
        count($this->workflowResult->getWarnings());

    $control->warning(
        sprintf(
            "Nested workflow '%s' emitted %s warning%s",
            $this->workflowResult->getWorkflowName(),
            $warnings,
            $warnings > 1 ? 's' : '',
        ),
    );
}
```

**Lógica**:

1. Cuenta warnings del workflow anidado
2. Reporta como warning del NestedWorkflow step
3. Aparece en logs del padre como warning del paso

### 11.2 Ejemplo de Warnings en Cascada

```
outer workflow:
  Warnings:
    - Nested workflow 'inner' emitted 1 warning
  
  inner workflow (anidado):
    Warnings:
      - Some processing issue
```

---

## 12. Implementación: uso con Construcción de Workflows

### 12.1 Caso Simple: Workflow Reutilizable

```php
// Definir un proceso estándar reutilizable
$payment_process = new Workflow('payment');
$payment_process
    ->validate(new ValidatePaymentData())
    ->process(new ProcessPayment())
    ->onError(new RollbackPayment());

// Usarlo en otros workflows
$checkout = new Workflow('checkout');
$checkout
    ->prepare(new LoadCart())
    ->process(
        new NestedWorkflow($payment_process),
        new CreateOrder(),
    )
    ->onSuccess(new SendConfirmation());

$subscription_renewal = new Workflow('renew-subscription');
$subscription_renewal
    ->prepare(new LoadSubscription())
    ->process(
        new NestedWorkflow($payment_process),  // Reutilizado
        new UpdateSubscriptionStatus(),
    );
```

**Beneficio**: `$payment_process` se compila una vez y se reutiliza múltiples veces.

### 12.2 Caso: Workflows con Lógica Condicional

```php
$approve_if_needed = new Workflow('conditional-approve');
$approve_if_needed
    ->process(
        new CheckApprovalRequirement(),
    )
    ->process(
        new NestedWorkflow(
            new Workflow('approval-flow')
                ->process(new RequestApproval())
                ->onSuccess(new NotifyApproved())
        ),
    );
```

---

## 13. Casos de Uso Prácticos

### 13.1 Microservice Integration

```php
$external_service = new Workflow('call-external-api');
$external_service
    ->process(
        new CallExternalAPI(),
        new MapExternalResponse(),
    );

$main_workflow = new Workflow('main');
$main_workflow
    ->process(
        new PrepareData(),
        new NestedWorkflow($external_service),
        new ProcessResult(),
    );
```

### 13.2 Multi-Stage Approval System

```php
$level_3 = new Workflow('executive-approval');
$level_3->process(new ExecutiveApprove());

$level_2 = new Workflow('manager-approval');
$level_2->process(
    new ManagerReview(),
    new NestedWorkflow($level_3),
);

$level_1 = new Workflow('team-lead-approval');
$level_1->process(
    new TeamLeadReview(),
    new NestedWorkflow($level_2),
);

// Uso
$result = $level_1->executeWorkflow($request);
```

### 13.3 ETL Pipeline with Stages

```php
$extract = new Workflow('extract');
$extract->process(new ExtractFromSource());

$transform = new Workflow('transform');
$transform->process(
    new NestedWorkflow($extract),
    new TransformData(),
);

$load = new Workflow('load');
$load->process(
    new NestedWorkflow($transform),
    new LoadToDatabase(),
);

// Full ETL
$result = $load->executeWorkflow();
```

---

## 14. Antipatrones y Limitaciones

### ❌ Antipatrón 1: Circular Nesting

```php
// MALO - ✗ Puede causar infinite recursion
$workflow_a = new Workflow('a');
$workflow_b = new Workflow('b');

$workflow_a->process(new NestedWorkflow($workflow_b));
$workflow_b->process(new NestedWorkflow($workflow_a));  // ✗ Circular

// Ejecución
$workflow_a->executeWorkflow();  // Infinite loop
```

**Solución**: No hagas referencias circulares. Usa únicamente herencia lineal.

### ❌ Antipatrón 2: Ignorar Success del Anidado

```php
// MALO - ✗ No revisas si el workflow anidado falló
$inner = new Workflow('inner');
// ... workflow definition ...

$outer_step = new class implements WorkflowStep {
    public function run($control, $container) {
        // ✗ Ejecutas y ignoras resultado
        $inner->executeWorkflow($container);
        // Continúas sin revisar éxito/fallo
    }
};

// BIEN - ✓ NestedWorkflow maneja esto automáticamente
$outer->process(new NestedWorkflow($inner));
```

### ❌ Antipatrón 3: Container Específico con Datos Conflictivos

```php
// MALO - ✗
$parent = new WorkflowContainer();
$parent->set('id', 'parent-id');

$nested_specific = new WorkflowContainer();
$nested_specific->set('id', 'nested-id');  // Conflicto

$nested = new NestedContainer($parent, $nested_specific);

// Resultado confuso:
$nested->get('id');  // 'nested-id' (nested gana)
// Pero set() escribe en ambos
$nested->set('id', 'new-value');
// Ahora parent->get('id') = 'new-value' (actualizado)
```

**Solución**: Usa namespaces en claves para evitar conflictos.

### ⚠️ Limitación 1: No hay Rollback Automático

```php
$nested->process(
    new NestedWorkflow(...),
    new DatabaseInsert(),
).onError(
    // ⚠️ Los datos del workflow anidado NO se revierten automáticamente
    new RollbackEverything(),
);
```

**Solución**: Implementa rollback explícitamente en OnError.

### ⚠️ Limitación 2: Nesting Depth sin Límite

```php
// Aunque técnicamente no hay límite...
$A->process(new NestedWorkflow($B));
$B->process(new NestedWorkflow($C));
$C->process(new NestedWorkflow($D));
// ... más niveles

// ⚠️ Stack se usa más, logging se hace mucho más profundo
// Considera limite práctico después de 3-4 niveles
```

---

## 15. Hallazgos Clave

### 15.1 Herencia vs Actualización Asimétrica

- **Lectura (`get()`)**: Cascada con prioridad al contenedor específico
- **Escritura (`set()`)**: Siempre escribe en AMBOS contenedores si existen
- **Resultado**: Datos suben automáticamente al padre

### 15.2 Propagación de Fallos por Etapa

El comportamiento de error depende del stage del workflow padre:

- **Pre-Process**: Aborta
- **Process**: Captura → OnError se ejecuta
- **Post-Process**: Captura sin abortar

### 15.3 Logging Automático de Nested

- Resultado completo del workflow anidado se adjunta
- Warnings del anidado se reportan automáticamente
- El log anidado se embebe completamente en el padre

### 15.4 Container Específico es Opcional

- Sin contenedor específico: hereda completamente del padre
- Con contenedor: permite datos específicos del workflow anidado
- `__call()` proporciona fallback para métodos personalizados

### 15.5 Reutilización Verdadera

- Mismo workflow puede anidarse múltiples veces
- Sin duplicación de lógica
- Cada instancia es estateless (no hay estado entre ejecuciones)

### 15.6 Composición sobre Herencia

- Los workflows anidados son pasos dentro de workflows
- Permite usar workflows como building blocks
- Ofrece mayor flexibilidad que herencia tradicional

---

## 16. Validación y Comportamiento

El comportamiento de workflows anidados se valida completo en tests:

- `testNestedWorkflow()`: Ejecución exitosa, herencia de datos
- `testNestedWorkflowFails()`: Propagación de errores, acceso a resultado
- `testNestedWorkflowHasMergedContainer()`: Comportamiento de NestedContainer con métodos personalizados

---

## 17. Conclusiones

El sistema de workflows anidados en php-workflow es **elegante y poderoso**:

### 17.1 Fortalezas

- **Reutilización pura**: Workflows existentes se componen sin modificación
- **Herencia de contexto**: Datos fluyen naturalmente entre niveles
- **Propagación automática**: Errores y warnings se reportan sin código boilerplate
- **Composición flexible**: Múltiples niveles de anidamiento sin límite práctico
- **Logging completo**: Jerarquía visible en debug output
- **Manejo de errores**: failStep automático si anidado falla

### 17.2 Patrones Emergentes

- **Building blocks**: Workflows simples se usan para construir complejos
- **Pipeline stages**: Cada etapa es un workflow reutilizable
- **Approval chains**: Aprobaciones multi-nivel por anidamiento
- **Microservice integration**: Workflows como adaptadores de servicios externos

### 17.3 Responsabilidades del Usuario

- Evitar referencias circulares
- Considerar nesting depth (práctico: máximo 3-4 niveles)
- Implementar rollback explícito si necesario
- Revisar logs para entender jerarquía

### 17.4 Integración Perfecta

- `NestedWorkflow` extiende `WorkflowStep` - se usa como cualquier paso
- `NestedContainer` extiende `WorkflowContainer` - adicionar transparente
- Propagación de estado automática en `set()` y cascada en `get()`
- Logging integrado con `StepInfo.NESTED_WORKFLOW`

---

## 18. Patrones de Diseño Recomendados

### 18.1 Patrón: Workflow Factory

```php
class WorkflowFactory {
    private $workflowCache = [];
    
    public function getPaymentWorkflow(): Workflow {
        if (!isset($this->workflowCache['payment'])) {
            $this->workflowCache['payment'] = new Workflow('payment');
            // ... configurar ...
        }
        return $this->workflowCache['payment'];
    }
    
    public function getCheckoutWorkflow(): Workflow {
        $checkout = new Workflow('checkout');
        $checkout->process(
            new NestedWorkflow($this->getPaymentWorkflow())
        );
        return $checkout;
    }
}
```

### 18.2 Patrón: Composable Workflows

```php
abstract class ComposableWorkflow {
    abstract public function build(): Workflow;
    
    public function nest(ComposableWorkflow $nested): Workflow {
        $workflow = $this->build();
        $workflow->process(new NestedWorkflow($nested->build()));
        return $workflow;
    }
}
```

---

## 19. Referencias de Código

### Ubicaciones Clave

- Core: `src/Step/NestedWorkflow.php` - Componente principal
- Container: `src/State/NestedContainer.php` - Herencia de contexto
- Base: `src/State/WorkflowContainer.php` - Contenedor base
- Tests: `tests/NestedWorkflowTest.php` - Todos los comportamientos validados

### Tests Recomendados

- `testNestedWorkflow()` - Reutilización básica, herencia de datos
- `testNestedWorkflowFails()` - Propagación de errores
- `testNestedWorkflowHasMergedContainer()` - Contenedor específico, métodos personalizados

---

## 20. Próximos Temas

Para profundizar:

- **Grupo 5: Middleware** - Los workflows anidados no aplican middleware de forma especial
- **Grupo 7: WorkflowResult** - Acceso a resultado completo del anidado
- **Grupo 12: Debugging** - Seguimiento de ejecución anidada
- **Casos de uso avanzados** en documentación de integración

