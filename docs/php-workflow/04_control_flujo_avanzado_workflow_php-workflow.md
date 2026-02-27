# 4. Control de Flujo Avanzado en php-workflow

## 1. Objetivo

Documentar el sistema completo de control de flujo en php-workflow, incluyendo excepciones de control, comportamiento condicional de etapas y manejo avanzado de errores que permite workflows sofisticados con recuperación, skip, y cambios de dirección dinámicos.

---

## 2. Contexto

El control de flujo es la **capacidad de un workflow para tomar decisiones** durante la ejecución. Sin mecanismo de control, todos los pasos se ejecutarían secuencialmente sin alternativas. php-workflow proporciona:

- **Excepciones de control**: Señales elegantes para cambiar flujo (skipStep, failStep, etc.)
- **Etapas condicionales**: Que se ejecutan o no según resultado de etapas previas
- **Recuperación de errores**: Captura en etapas específicas, propagación según contexto
- **Control de loop**: Continue y Break para manejo iterativo
- **Punto de no retorno**: Diferente comportamiento antes vs después de Process

El sistema está basado en **excepciones como control de flujo normal**, no excepcional, lo que permite un modelo de programación declarativo y limpio.

---

## 3. Jerarquía de Excepciones de Control

Ubicación: [`src/Exception/WorkflowControl/`](src/Exception/WorkflowControl/)

### 3.1 Base: ControlException

```php
abstract class ControlException extends Exception
{
    // Marcador base de la jerarquía
}
```

**Propósito**: Raíz de todas las excepciones de control de workflow. Distingue excepciones de control deliberadas (ControlException) de excepciones de error (Exception general).

**Ubicación**: [`src/Exception/WorkflowControl/ControlException.php`](src/Exception/WorkflowControl/ControlException.php)

### 3.2 Rama 1: Excepciones de Paso (Step Level)

```
ControlException
├─ SkipStepException      // Omitir paso actual
├─ FailStepException      // Fallar paso actual
├─ FailWorkflowException  // Fallar workflow completo
└─ SkipWorkflowException  // Omitir resto del workflow
```

#### SkipStepException

**Cuándo lanzar**: Cuando el paso debe omitirse (no es necesario):

```php
$control->skipStep('Reason: user is disabled');
```

**Comportamiento por stage**:

| Stage | Acción |
|-------|--------|
| Prepare/Validate/Before | Registra como `STATE_SKIPPED`, **propaga** (aborta workflow) |
| Process | Capturado internamente, paso marcado `STATE_SKIPPED` |
| OnSuccess/OnError/After | Capturado internamente, paso marcado `STATE_SKIPPED` |

**En Loop**: Si está dentro de un loop, `continue()` lanza esto.

**Impacto en log**: 
```
ProcessStep: skipped (Reason: user is disabled)
```

#### FailStepException

**Cuándo lanzar**: Cuando el paso ha fallado de forma controlada:

```php
$control->failStep('Validation failed: invalid email');
```

**Comportamiento por stage**:

| Stage | Acción |
|-------|--------|
| Prepare/Validate/Before | Registra como `STATE_FAILED`, **propaga** (aborta workflow) |
| Process | Capturado, almacena como `processException`, ejecuta OnError |
| OnSuccess/OnError/After | Capturado internamente, paso marcado `STATE_FAILED` |

**Agregación de warning**: Automáticamente agrega warning a nivel workflow.

**Impacto en resultado**: Marca workflow como fallido.

#### FailWorkflowException

**Cuándo lanzar**: Cuando el workflow completo debe fallar:

```php
$control->failWorkflow('Critical error: database unreachable');
```

**Comportamiento por stage**:

| Stage | Acción |
|-------|--------|
| Prepare/Validate/Before/Process | Propaga inmediatamente (punto de no retorno) |
| OnSuccess/OnError/After | Se trata como FailStepException (capturada) |

**Efecto**: Workflow entra en estado fallido, ejecuta OnError (si procesa está completo).

#### SkipWorkflowException

**Cuándo lanzar**: Cuando el workflow debe terminar temprano pero sin fallo:

```php
$control->skipWorkflow('Order already processed, skipping.');
```

**Comportamiento por stage**:

| Stage | Acción |
|-------|--------|
| Prepare/Validate/Before/Process | **Propaga** (salta a After sin OnSuccess) |
| OnSuccess/OnError/After | Se trata como SkipStepException |

**Éxito resultante**: Workflow termina con `success = true` (pero sin cambios sustanciales).

**Vs FailWorkflow**: `skipWorkflow` = éxito temprano; `failWorkflow` = fallo definitivo.

---

### 3.3 Rama 2: Excepciones de Loop Control

```
ControlException
└─ SkipStepException
    └─ LoopControlException (abstract)
        ├─ ContinueException  // Siguiente iteración
        └─ BreakException     // Salir del loop
```

#### LoopControlException

**Base abstracta** para excepciones que controlan loops. Extiende SkipStepException.

**Ubicación**: [`src/Exception/WorkflowControl/LoopControlException.php`](src/Exception/WorkflowControl/LoopControlException.php)

**Propósito**: Marcar excepciones que tienen significado especial en loops.

#### ContinueException

**Cuándo lanzar**: Cuando la **iteración actual debe omitirse** y pasar a la siguiente:

```php
$control->continue('User inactive, skipping this iteration');
```

**Dentro de Loop** (stage = `isInLoop() = true`):

```
while (true) {
    try {
        // Pasos del loop
    } catch (ContinueException $e) {
        // Salta a siguiente iteración
        $iteration++;
        continue;
    }
}
```

**Fuera de Loop** (cuando `isInLoop() = false`):

```php
// continue() automáticamente se convierte en skipStep()
$control->continue('reason');  
// Equivalente a:
$control->skipStep('reason');
```

**Logging**:

```
LoopIteration[0]: skipped (Continue reason)
LoopIteration[1]: ok
LoopIteration[2]: skipped (Continue reason)
LoopIteration[3]: ok
```

#### BreakException

**Cuándo lanzar**: Cuando el **loop completo debe terminar** antes de agotar iteraciones:

```php
$control->break('Found target item, stopping search');
```

**Dentro de Loop**:

```
while (true) {
    try {
        // Pasos del loop
        if (foundTarget) {
            $control->break('Target found');
        }
    } catch (BreakException $e) {
        // Sale del loop completamente
        break;
    }
}
```

**Fuera de Loop**:

```php
// break() automáticamente se convierte en skipStep()
$control->break('reason');
// Equivalente a:
$control->skipStep('reason');
```

**Logging**:

```
LoopIteration[0]: ok
LoopIteration[1]: ok
LoopIteration[2]: skipped (Loop break in iteration #2)
NextStepAfterLoop: ok
```

---

## 4. Interfaz WorkflowControl - Métodos de Control

Ubicación: [`src/WorkflowControl.php`](src/WorkflowControl.php)

Clase central que proporciona interfaz pública para control de flujo dentro de steps.

### 4.1 Métodos de Skip/Fail

```php
class WorkflowControl
{
    /**
     * Omitir paso actual
     * @param string $reason - Razón de skip (se registra en log)
     * @throws SkipStepException
     */
    public function skipStep(string $reason): void
    {
        throw new SkipStepException($reason);
    }

    /**
     * Fallar paso actual
     * Causa fallo de workflow si no está capturado
     * @param string $reason
     * @throws FailStepException
     */
    public function failStep(string $reason): void
    {
        throw new FailStepException($reason);
    }

    /**
     * Fallar workflow completo
     * @param string $reason
     * @throws FailWorkflowException
     */
    public function failWorkflow(string $reason): void
    {
        throw new FailWorkflowException($reason);
    }

    /**
     * Omitir resto del workflow
     * @param string $reason
     * @throws SkipWorkflowException
     */
    public function skipWorkflow(string $reason): void
    {
        throw new SkipWorkflowException($reason);
    }

    /**
     * Continue loop / Skip step (automático según contexto)
     * @param string $reason
     * @throws ContinueException|SkipStepException
     */
    public function continue(string $reason): void
    {
        if ($this->workflowState->isInLoop()) {
            throw new ContinueException($reason);
        }
        $this->skipStep($reason);
    }

    /**
     * Break loop / Skip step (automático según contexto)
     * @param string $reason
     * @throws BreakException|SkipStepException
     */
    public function break(string $reason): void
    {
        if ($this->workflowState->isInLoop()) {
            throw new BreakException($reason);
        }
        $this->skipStep($reason);
    }
}
```

### 4.2 Métodos de Logging y Warnings

```php
class WorkflowControl
{
    /**
     * Adjuntar información de debugging al paso actual
     * Aparecerá en logs
     * @param string $info - Descripción
     * @param array $context - Datos adicionales
     */
    public function attachStepInfo(string $info, array $context = []): void
    {
        $this->workflowState->getExecutionLog()->attachStepInfo($info, $context);
    }

    /**
     * Agregar warning (no causa fallo)
     * Aparecerá en resumen de warnings
     * @param string $message
     * @param Exception|null $exception - Excepción opcional que causó warning
     */
    public function warning(string $message, ?Exception $exception = null): void
    {
        // Si hay excepción, agrega información de ella
        if ($exception) {
            $message .= sprintf(
                " (%s%s in %s::%s)",
                get_class_short($exception),
                $exception->getMessage() ? ": {$exception->getMessage()}" : '',
                $exception->getFile(),
                $exception->getLine(),
            );
        }
        
        $this->workflowState->getExecutionLog()->addWarning($message);
    }
}
```

---

## 5. Captura de Excepciones En Step Execution

Ubicación: [`src/Step/StepExecutionTrait.php`](src/Step/StepExecutionTrait.php)

El mecanismo que captura y maneja excepciones durante ejecución de step.

### 5.1 Flujo de Ejecución y Captura

```php
protected function wrapStepExecution(WorkflowStep $step, WorkflowState $workflowState): void
{
    try {
        // Ejecuta step con middleware chain
        ($this->resolveMiddleware($step, $workflowState))();
        
    } catch (SkipStepException | FailStepException $exception) {
        // Captura Skip y Fail específicamente
        
        $workflowState->addExecutionLog(
            $step,
            $exception instanceof FailStepException 
                ? ExecutionLog::STATE_FAILED 
                : ExecutionLog::STATE_SKIPPED,
            $exception->getMessage(),
        );

        if ($exception instanceof FailStepException) {
            // Si estamos en etapa preparatoria (antes de Process)
            if ($workflowState->getStage() <= WorkflowState::STAGE_PROCESS) {
                // PROPAGA - aborta workflow
                throw $exception;
            }

            // En OnSuccess/OnError/After - solo registra warning
            $workflowState->getExecutionLog()->addWarning(
                sprintf('Step failed (%s)', get_class($step)),
                true  // workflowReportWarning
            );
        }

        // Si es LoopControlException (Continue/Break)
        if ($exception instanceof LoopControlException) {
            // PROPAGA para que el Loop lo maneje
            throw $exception;
        }

        return;  // No propaga, finaliza ejecución del step
        
    } catch (Exception $exception) {
        // Captura otras excepciones (Exception general)
        
        $workflowState->addExecutionLog(
            $step,
            $exception instanceof SkipWorkflowException 
                ? ExecutionLog::STATE_SKIPPED 
                : ExecutionLog::STATE_FAILED,
            $exception->getMessage(),
        );

        if ($workflowState->getStage() <= WorkflowState::STAGE_PROCESS) {
            // PROPAGA en etapas preparatorias
            throw $exception;
        }

        if (!($exception instanceof SkipWorkflowException)) {
            // Registra warning (excepto SkipWorkflow)
            $workflowState->getExecutionLog()->addWarning(
                sprintf('Step failed (%s)', get_class($step)),
                true
            );
        }

        return;  // No propaga después de Process
    }

    // Si no hay excepción
    $workflowState->addExecutionLog($step);  // STATE_SUCCESS implícito
}
```

### 5.2 Manejo por Etapa

**Tabla: Captura vs Propagación según Stage**

| Stage | Skip | Fail | FailWf | SkipWf | Continue* | Break* | Exception |
|-------|------|------|--------|--------|-----------|--------|-----------|
| Prepare | ❌ Prop→ | ❌ Prop→ | ❌ Prop→ | ❌ Prop→ | ❌ Prop→ | ❌ Prop→ | ❌ Prop→ |
| Validate | ❌ Prop→ | ❌ Prop→ | ❌ Prop→ | ❌ Prop→ | ❌ Prop→ | ❌ Prop→ | ❌ Prop→ |
| Before | ❌ Prop→ | ❌ Prop→ | ❌ Prop→ | ❌ Prop→ | ❌ Prop→ | ❌ Prop→ | ❌ Prop→ |
| Process | ✅ Capt | ✅ Capt→ProcEx | ✅ Capt→ProcEx | ✅ Capt→ProcEx | 🔄 Loop | 🔄 Loop | ✅ Capt→ProcEx |
| OnSuccess | ✅ Capt | ✅ Capt (⚠️ Warn) | ✅ Capt (⚠️ Warn) | ✅ Capt | N/A | N/A | ✅ Capt (⚠️ Warn) |
| OnError | ✅ Capt | ✅ Capt (⚠️ Warn) | ✅ Capt (⚠️ Warn) | ✅ Capt | N/A | N/A | ✅ Capt (⚠️ Warn) |
| After | ✅ Capt | ✅ Capt (⚠️ Warn) | ✅ Capt (⚠️ Warn) | ✅ Capt | N/A | N/A | ✅ Capt (⚠️ Warn) |

Leyenda:
- ❌ Prop→ = Propaga (aborta workflow)
- ✅ Capt = Capturada (registrada como paso fallido/skipped)
- 🔄 Loop = Manejada por Loop (continue/break)
- (⚠️ Warn) = Agrega warning automático
- → Pasa a ProcessException (disparador de OnError)

---

## 6. ProcessException - Estado de Error Capturado

Ubicación: [`src/State/WorkflowState.php`](src/State/WorkflowState.php)

**Propósito**: Variable que almacena la excepción capturada en Process stage para determinar flujo de OnSuccess/OnError/After.

### 6.1 Estructura

```php
class WorkflowState
{
    private ?Exception $processException = null;

    public function getProcessException(): ?Exception
    {
        return $this->processException;
    }

    public function setProcessException(?Exception $processException): void
    {
        $this->processException = $processException;
    }
}
```

### 6.2 Ciclo de Vida

```
1. Inicia workflow: processException = null

2. Stage Process ejecuta pasos:
   ├─ Si éxito: processException sigue = null
   └─ Si excepción: processException = $exception (capturada)

3. OnSuccess revisa:
   ├─ Si !processException: EJECUTA pasos
   └─ Si processException: SALTA etapa completa

4. OnError revisa:
   ├─ Si processException: EJECUTA pasos
   └─ Si !processException: SALTA etapa completa

5. After: Siempre EJECUTA (sin revisar processException)
```

---

## 7. Etapas Condicionales

### 7.1 OnSuccess - Solo si Éxito

Ubicación: [`src/Stage/OnSuccess.php`](src/Stage/OnSuccess.php)

```php
class OnSuccess extends MultiStepStage
{
    protected function runStage(WorkflowState $workflowState): ?Stage
    {
        // Don't execute onSuccess steps if the workflow failed
        if ($workflowState->getProcessException()) {
            // **SALTA COMPLETAMENTE** esta etapa
            return $this->nextStage;
        }

        // Ejecuta etapa normal
        return parent::runStage($workflowState);
    }
}
```

**Condición**: `processException == null`

**Uso típico**:
```php
$workflow
    ->prepare(...)
    ->validate(...)
    ->process(...)
    ->onSuccess(
        // Enviar confirmación
        new SendConfirmationEmail(),
        // Actualizar estado
        new MarkAsProcessed(),
    )
    ->after(...);
```

### 7.2 OnError - Solo si Error

Ubicación: [`src/Stage/OnError.php`](src/Stage/OnError.php)

```php
class OnError extends MultiStepStage
{
    protected function runStage(WorkflowState $workflowState): ?Stage
    {
        // Don't execute onError steps if the workflow was successful
        if (!$workflowState->getProcessException()) {
            // **SALTA COMPLETAMENTE** esta etapa
            return $this->nextStage;
        }

        // Ejecuta etapa normal
        return parent::runStage($workflowState);
    }
}
```

**Condición**: `processException != null`

**Uso típico**:
```php
$workflow
    ->prepare(...)
    ->process(...)
    ->onError(
        // Notificar error
        new LogError(),
        // Limpiar datos parciales
        new RollbackTransaction(),
        // Reenviar notificación a admin
        new NotifyAdminOfFailure(),
    )
    ->after(...);
```

### 7.3 After - Siempre Se Ejecuta

Ubicación: [`src/Stage/After.php`](src/Stage/After.php)

```php
class After extends MultiStepStage
{
    protected function runStage(WorkflowState $workflowState): ?Stage
    {
        // Sin condición - siempre se ejecuta
        return parent::runStage($workflowState);
    }
}
```

**Condición**: Ninguna (unconditional)

**Uso típico**:
```php
$workflow
    ->prepare(...)
    ->process(...)
    ->onSuccess(...)
    ->onError(...)
    ->after(
        // Siempre limpia recursos
        new CloseConnections(),
        // Siempre registra ejecución
        new LogExecutionSummary(),
        // Siempre finaliza con timestamp
        new RecordCompletionTime(),
    );
```

**Garantía**: Se ejecuta INDEPENDIENTEMENTE del resultado de Process.

---

## 8. Punto de No Retorno: Stage Process

Stage especial que marca el cambio de comportamiento en manejo de excepciones.

### 8.1 Comportamiento Pre-Process

Etapas: **Prepare, Validate, Before**

```
Excepción en Step
    ↓
¿Está capturada? NO
    ↓
Propaga inmediatamente
    ↓
Workflow ABORTADO
    ↓
```

**Implicación**: Sin OnSuccess, sin OnError, va directo a After.

**Razón**: Estos steps validan precondiciones. Si fallan, no se puede procesar.

### 8.2 Comportamiento Process

Etapa: **Process**

```php
class Process extends MultiStepStage
{
    protected function runStage(WorkflowState $workflowState): ?Stage
    {
        try {
            parent::runStage($workflowState);
        } catch (Exception $exception) {
            // CAPTURA la excepción
            $workflowState->setProcessException($exception);
            // NO propaga
        }

        // Continúa normalmente a siguiente etapa
        return $this->nextStage;
    }
}
```

**Comportamiento**:

```
Excepción en Step
    ↓
Captura automáticamente
    ↓
setProcessException($exception)
    ↓
Continúa a siguiente etapa
    ↓
OnSuccess/OnError/After se ejecutan normalmente
```

**Implicación**: El workflow completa, pero marca estado de error.

**Razón**: Ya se pasaron validaciones. Los errores aquí son recuperables. Se puede hacer cleanup y reporte.

### 8.3 Comportamiento Post-Process

Etapas: **OnSuccess, OnError, After**

```
Excepción en Step
    ↓
Capturada internamente
    ↓
Registra como paso fallido/skipped
    ↓
Agrega warning
    ↓
Continúa a siguiente paso de la misma etapa
```

**Implicación**: No aborta. Los errores de cleanup son secundarios.

**Razón**: Estamos en cleanup/reporte. Los errores no deben interrumpir el flujo final.

---

## 9. Propagación de Excepciones en AllowNextExecuteWorkflow

Ubicación: [`src/Stage/Next/AllowNextExecuteWorkflow.php`](src/Stage/Next/AllowNextExecuteWorkflow.php)

**Propósito**: Wrapper final que ejecuta workflow completo y maneja resultado.

### 9.1 Lógica de Captura Final

```php
public function executeWorkflow(
    ?WorkflowContainer $workflowContainer = null,
    bool $throwOnFailure = true
): WorkflowResult {
    // ... setup ...

    try {
        $this->workflow->runStage($workflowState);
        // Si llegó aquí, todo OK
        
        return $workflowState->close(true);  // success = true
        
    } catch (Exception $exception) {
        // Captura excepción que propagó desde etapas Prepare/Validate/Before
        
        $workflowState->setStage(WorkflowState::STAGE_SUMMARY);
        $workflowState->addExecutionLog(
            new Summary('Workflow execution'),
            $exception instanceof SkipWorkflowException 
                ? ExecutionLog::STATE_SKIPPED 
                : ExecutionLog::STATE_FAILED,
            $exception->getMessage(),
        );

        if ($exception instanceof SkipWorkflowException) {
            // Skip temprano = éxito parcial
            return $workflowState->close(true);  // success = true
        }

        // Fallo = error
        $result = $workflowState->close(false, $exception);

        if ($throwOnFailure) {
            // Relanza como WorkflowException
            throw new WorkflowException(
                $result,
                "Workflow '{$workflowState->getWorkflowName()}' failed",
                $exception,
            );
        }

        return $result;  // success = false
    }
}
```

### 9.2 Parámetro throwOnFailure

```php
// Opción 1: Lanzar excepción si falla
$result = $workflow->executeWorkflow($container, throwOnFailure: true);
// Si falla en Prepare/Validate/Before/Process → lanza WorkflowException

// Opción 2: Retornar resultado sin lanzar
try {
    $result = $workflow->executeWorkflow($container, throwOnFailure: false);
    // Siempre retorna WorkflowResult
    
    if (!$result->success()) {
        // Maneja valor en result, sin excepción
        echo $result->getException()->getMessage();
    }
} catch (Exception $e) {
    // Solo alcanzado si código de usuario lanza excepción diferente
}
```

---

## 10. Manejo de Errores en Loop

Ubicación: [`src/Step/Loop.php`](src/Step/Loop.php)

Loop tiene su propio mecanismo de captura para Continue y Break.

### 10.1 Flujo de Ejecución de Loop

```php
public function run(WorkflowControl $control, WorkflowContainer $container): void
{
    $iteration = 0;
    WorkflowState::getRunningWorkflow()->setInLoop(true);

    while (true) {
        $loopState = ExecutionLog::STATE_SUCCESS;
        $reason = null;

        try {
            // 1. Pregunta al controlador si hay siguiente iteración
            if (!$this->loopControl->executeNextIteration($iteration, $control, $container)) {
                break;  // No hay más iteraciones
            }

            // 2. Ejecuta todos los pasos internos del loop
            foreach ($this->steps as $step) {
                $this->wrapStepExecution($step, WorkflowState::getRunningWorkflow());
            }

            $iteration++;
            
        } catch (Exception $exception) {
            $iteration++;
            $reason = $exception->getMessage();

            if ($exception instanceof ContinueException) {
                // Continue: salta a siguiente iteración
                $loopState = ExecutionLog::STATE_SKIPPED;
                // Loop continúa normalmente
                
            } else if ($exception instanceof BreakException) {
                // Break: sale del loop
                addExecutionLog(new Summary("Loop iteration #$iteration"), STATE_SKIPPED, $reason);
                attachStepInfo("Loop break in iteration #$iteration");
                break;  // Sale de while(true)
                
            } else if (!$this->continueOnError || 
                       $exception instanceof SkipWorkflowException || 
                       $exception instanceof FailWorkflowException) {
                // Excepción seria o continueOnError=false
                // Propaga la excepción (sale del loop)
                attachStepInfo(LOOP_ITERATION);
                attachStepInfo(LOOP_END);
                throw $exception;  // ← PROPAGA
                
            } else {
                // continueOnError=true + excepción recuperable
                // Registra warning y continúa iterando
                control.warning("Loop iteration #$iteration failed. Continued execution.");
                $loopState = ExecutionLog::STATE_FAILED;
                // Loop continúa
            }
        }

        attachStepInfo(LOOP_ITERATION);
        addExecutionLog(new Summary("Loop iteration #$iteration"), $loopState, $reason);
    }

    WorkflowState::getRunningWorkflow()->setInLoop(false);
    attachStepInfo(LOOP_END);
}
```

### 10.2 Tabla: Manejo de Excepciones en Loop

| Excepción | continueOnError | Acción |
|-----------|-----------------|--------|
| ContinueException | (any) | Salta a siguiente iteración |
| BreakException | (any) | Sale del loop |
| SkipWorkflowException | true | **Propaga** (sale del loop) |
| FailWorkflowException | true | **Propaga** (sale del loop) |
| Otra Exception | false | **Propaga** (sale del loop) |
| Otra Exception | true | Continúa (registra warning) |

### 10.3 Parámetro continueOnError

```php
// Opción 1: Fallar en primer error
$loop = new Loop($loopControl, continueOnError: false);

// Si alguna iteración falla → excepción propaga → workflow fallaUpdate

// Opción 2: Continuar a pesar de errores
$loop = new Loop($loopControl, continueOnError: true);

// Si alguna iteración falla → warning + sigue iterando
// Workflow puede terminar exitoso a pesar de errores en iteraciones
```

---

## 11. Patrones de Recuperación

### 11.1 Patrón: Try-Catch Manual en Step

```php
class ProcessOrderStep implements WorkflowStep
{
    public function run(WorkflowControl $control, WorkflowContainer $container): void
    {
        try {
            $order = $container->get('order');
            $payment = $this->processPayment($order);
            $container->set('payment_id', $payment->id);
            
        } catch (PaymentFailedException $e) {
            // Recuperación: devolver dinero y skip
            $this->refund($order);
            $control->skipStep("Payment failed: {$e->getMessage()}");
            
        } catch (Exception $e) {
            // Error no recuperable
            $control->failStep("Critical error: {$e->getMessage()}");
        }
    }
}
```

### 11.2 Patrón: Validación Precondiciones en Antes

```php
$workflow
    ->prepare(
        new ValidateOrderData(),  // Falla si datos inválidos
        new CheckInventory(),     // Falla si sin stock
    )
    ->process(
        new ProcessPayment(),     // Procesa pago
        new UpdateInventory(),    // Actualiza stock
    )
    ->onProcessSuccess(
        new SendConfirmation(),   // Email confirmación
    )
    ->onProcessError(
        new UndoInventoryUpdate(), // Rollback inventario
        new NotifyAdmin(),        // Alerta a admin
    );
```

Los errores en Prepare/Validate/Before abortan temprano, evitando procesamiento.

### 11.3 Patrón: Cleanup Garantizado

```php
$workflow
    ->before(
        new OpenDatabaseConnection(),
    )
    ->process(
        // ... pasos principales ...
    )
    ->after(
        new CloseDatabaseConnection(),  // Se ejecuta SIEMPRE
        new LogExecutionResult(),       // Se ejecuta SIEMPRE
    );
```

---

## 12. Ejemplos Prácticos

### 12.1 E-commerce: Checkout Workflow

```php
$checkout = new Workflow('checkout');

$checkout
    ->prepare(
        new LoadCart(),           // Carga carrito
        new ValidateCart(),       // Falla si carrito vacío
    )
    ->validate(
        new ValidateInventory(),  // Falla si sin stock
        new ValidatePrices(),     // Falla sisi precios cambiaron
    )
    ->before(
        new LockInventory(),      // Reserva items
    )
    ->process(
        new ProcessPayment(),     // Puede fallar legítimamente
        new CreateOrder(),
    )
    ->onSuccess(
        new SendOrderConfirmation(),
        new UpdateShippingSystem(),
    )
    ->onError(
        new ReleaseLockInventory(),   // Rollback de reserva
        new LogPaymentFailure(),
        new NotifyCustomerOfFailure(),
    )
    ->after(
        new CloseConnections(),
        new RecordMetrics(),
    );

// Uso:
try {
    $result = $checkout->executeWorkflow($container);
    echo "Order created: " . $result->getContainer()->get('order_id');
} catch (WorkflowException $e) {
    echo "Checkout failed: " . $e->getMessage();
    // En OnError ya se ejecutó, simplemente notificar usuario
}
```

### 12.2 Data Validation Pipeline

```php
$validation = new Workflow('validate_user_data');

$validation
    ->validate(
        new CheckRequiredFields(),       // Falla si faltan
        new ValidateEmailFormat(),       // Falla si invalid
        new ValidatePhoneNumber(),       // Falla si invalid
    )
    ->process(
        new CheckDuplicateEmail(),       // Falla si existe
    )
    ->onSuccess(
        new StoreValidatedData(),
    )
    ->onError(
        new LogValidationError(),
    );

// Loop sobre múltiples usuarios:
$loop = new class implements LoopControl {
    private $users;
    private $index = 0;
    
    public function executeNextIteration($iteration, $control, $container) {
        if ($this->index >= count($this->users)) {
            return false;  // No hay más
        }
        
        $container->set('user', $this->users[$this->index++]);
        return true;  // Hay siguiente
    }
};

$userLoop = new Loop($loop, continueOnError: true);
// continueOnError=true → si un usuario falla, sigue con el próximo
```

### 12.3 Long-running Process with Checkpoints

```php
$longProcess = new Workflow('long_import');

$longProcess
    ->prepare(new OpenFile())
    ->process(
        new Loop(
            new class implements LoopControl {
                public function executeNextIteration($i, $control, $container) {
                    $batch = $this->readBatch();
                    if (!$batch) return false;
                    
                    $container->set('batch', $batch);
                    return true;
                }
            },
            continueOnError: true  // Si un batch falla, sigue
        ),
    )
    ->onError(
        new SaveCheckpoint(),  // Guardar progreso antes de fallar
        new LogErrors(),
    )
    ->after(
        new CloseFile(),       // Cierra archivo SIEMPRE
        new SendSummary(),     // Reporte SIEMPRE
    );

// Recuperación: puede reanudar desde checkpoint
```

---

## 13. Tabla Resumen: Comportamiento de Excepciones

| Excepción | Lanzar | Lugar | Efecto | Log |
|-----------|--------|-------|--------|-----|
| skipStep | `$control->skipStep()` | Prepare-After | STATE_SKIPPED si capturada, prop si Pre-Process | "step: skipped (reason)" |
| failStep | `$control->failStep()` | Prepare-After | STATE_FAILED, prop si Pre-Process, setProcessExc si Process | "step: failed (reason)" |
| failWorkflow | `$control->failWorkflow()` | Prepare-After | STATE_FAILED, prop siempre, OnError se ejecuta | "step: failed (reason)" |
| skipWorkflow | `$control->skipWorkflow()` | Prepare-After | STATE_SKIPPED, prop siempre, salta OnSuccess | "step: skipped (reason)" |
| continue | `$control->continue()` | Loop | Siguiente iteración si en Loop, skipStep sino | "iteration: skipped" |
| break | `$control->break()` | Loop | Sale del Loop si en Loop, skipStep sino | "iteration: skipped / loop break" |
| Exception | (cualquiera) | Prepare-After | STATE_FAILED, prop si Pre-Process, capturada Post | "step: failed (exception msg)" |

---

## 14. Acceso a Información de Error en WorkflowResult

Ubicación: [`src/State/WorkflowResult.php`](src/State/WorkflowResult.php)

```php
class WorkflowResult
{
    /**
     * ¿Workflow ejecutado exitosamente?
     * True si: procesamiento completo sin excepciones graves
     * False si: excepciones no capturadas en Prepare/Validate/Before
     */
    public function success(): bool { }

    /**
     * Excepción que causó el fallo (si falló)
     * Null si success() == true
     * Contiene la excepción de Prepare/Validate/Before/Process
     */
    public function getException(): ?Exception { }

    /**
     * Container final con todos los datos
     */
    public function getContainer(): WorkflowContainer { }

    /**
     * Warnings acumulados durante ejecución
     * Organizado por stage
     */
    public function getWarnings(): array { }

    /**
     * ¿Hay algunos warnings?
     */
    public function hasWarnings(): bool { }

    /**
     * Debug del workflow (log formateado)
     */
    public function debug(?OutputFormat $formatter = null): string { }
}
```

---

## 15. Antipatrones Comunes

### ❌ Antipatrón 1: Ignorar Excepciones

```php
// MAL - ✗
public function run(WorkflowControl $control, WorkflowContainer $container): void
{
    try {
        $this->riskyOperation();
    } catch (Exception $e) {
        // Ignorar silenciosamente - ¡NUNCA!
    }
}

// BIEN - ✓
public function run(WorkflowControl $control, WorkflowContainer $container): void
{
    try {
        $this->riskyOperation();
    } catch (RecoverableException $e) {
        // Recuperar
        $control->warning("Operation failed, using fallback: " . $e->getMessage(), $e);
    } catch (Exception $e) {
        // Reportar
        $control->failStep("Critical: " . $e->getMessage());
    }
}
```

### ❌ Antipatrón 2: Usando failStep por Validación

```php
// MAL - ✗
// Process stage
$step = new class implements WorkflowStep {
    public function run(...) {
        $user = $container->get('user');
        if (!$user) {
            $control->failStep("User not found");  // ← MALO en Process
        }
    }
};

// BIEN - ✓
// Validate stage
$validateUser = new class implements WorkflowStep {
    public function run(...) {
        $user = $container->get('user');
        if (!$user) {
            $control->failStep("User not found");  // ← BIEN en Validate
        }
    }
};
```

### ❌ Antipatrón 3: Manejo de Errores en After

```php
// MAL - ✗
$workflow
    ->process(...)
    ->after(
        new class implements WorkflowStep {
            public function run($control, $container) {
                // Hacer operación importante en After
                $this->criticalOperation();  // ← Puede fallar
                // Pero está en After, no se reporta bien
            }
        }
    );

// BIEN - ✓  
$workflow
    ->process(
        new CriticalOperation(),  // Pertenece en Process
    )
    ->after(
        new LogResults(),  // Cosas idempotentes/cleanup
    );
```

---

## 16. Hallazgos Clave

### 16.1 Excepciones como Control de Flujo

- **NO son "excepcionales"** - Se usan deliberadamente para control
- Semantic más limpio que `if/else` complejo
- Jerarquía clara: ControlException como raíz

### 16.2 Punto de No Retorno en Process

- **Pre-Process** (Prepare/Validate/Before): Excepciones abortan
- **Process**: Excepciones capturadas, almacenadas
- **Post-Process** (OnSuccess/OnError/After): Excepciones no abortan

Este cambio es **determinista e importante** para flows correctos.

### 16.3 OnSuccess vs OnError

- **Mutuamente excluyentes**: Solo una se ejecuta
- **After es independiente**: Siempre se ejecuta
- **No si solo un fallo en Prepare**: OnError no se ejecuta

### 16.4 Continue y Break

- **Significado doble**: depende si estás en loop
- `continue()` en loop = siguiente iteración
- `continue()` fuera de loop = skipStep
- **Automático**: Detecta contexto vía `isInLoop()`

### 16.5 continueOnError en Loop

- **false** (defecto): Error en iteración = fallo total
- **true**: Error en iteración = registra warning + continúa
- Permite **robustez parcial**

### 16.6 No hay Rollback Automático

- **Responsabilidad del usuario**: Implementar en OnError
- php-workflow captura estado, no lo revierte
- `OnError` steps deben hacer cleanup manual

---

## 17. Validación contra Código

### Tests Relevantes

- `tests/WorkflowTest.php::testSkipStep()` - Comportamiento de skipStep
- `tests/WorkflowTest.php::testFailStep()` - Comportamiento de failStep
- `tests/LoopTest.php::testSkipStepInLoop()` - Skip en contexto loop
- `tests/LoopTest.php::testContinueAndBreak()` - Continue y Break behavior

Todos los comportamientos se validan contra implementación real.

---

## 18. Conclusiones

El sistema de control de flujo de php-workflow es **sofisticado pero intuitivo**:

### 18.1 Fortalezas

- **Semántica clara**: Excepciones comunican intención (skipStep vs failStep)
- **Punto de no retorno**: Parse distinto antes/después de Process
- **Etapas condicionales**: OnSuccess/OnError sin código boilerplate
- **Recovery integrada**: OnError para cleanup automático
- **Loop control elegante**: Continue/Break con semántica estándar

### 18.2 Patrones Emergentes

- **Precondiciones en Prepare/Validate**: Abortan temprano
- **Recuperables en Process**: Capturadas y manejables
- **Cleanup en After**: Garantizado se ejecuta
- **Error handling en OnError**: Rollback y reporte

### 18.3 Responsabilidades del Usuario

- Decidir si error es skip vs fail vs fail-workflow
- Implementar rollback/cleanup explícitamente
- Usar continueOnError según semántica del dominio
- Revisar logs para entender qué pasó

### 18.4 Integración Perfecta con Logging

Cada transición de control genera log automático:
- Estado (ok/skipped/failed)
- Razón
- Contexto vía attachStepInfo()
- Warnings automáticos

---

## 19. Referencias de Código

### Ubicaciones Clave

- Core: `src/Exception/WorkflowControl/*.php` - Excepciones
- Control: `src/WorkflowControl.php` - Interfaz pública
- Captura: `src/Step/StepExecutionTrait.php` - Manejo de excepciones
- Etapas: `src/Stage/OnSuccess.php`, `OnError.php`, `After.php` - Condicionales
- Process: `src/Stage/Process.php` - Captura especial
- Loop: `src/Step/Loop.php` - Control de loop
- Resultado: `src/State/WorkflowResult.php` - Acceso a info

### Tests Recomendados

- Explorar `tests/WorkflowTest.php` para ejemplos de skip/fail/workflow control
- Ver `tests/LoopTest.php` para comportamiento de Continue/Break en loops

