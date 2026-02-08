# 7. Loops y LoopControl en php-workflow

## 1. Objetivo

Documentar el sistema de loops que permite repetir un conjunto de pasos múltiples veces bajo control programático, incluyendo mecanismos para continuar, romper, saltar iteraciones y manejar errores dentro de ciclos.

---

## 2. Contexto

Frecuentemente necesitas procesar **colecciones de datos** o **repetir lógica** hasta que se cumpla una condición:

- Procesar cada item de una lista
- Reintentar un paso hasta n intentos
- Procesar filas de un archivo batch
- Iteración hasta completitud
- Tareas generadoras de trabajo

php-workflow proporciona un mecanismo **loop-aware** que:
- Integra loops como WorkflowStep dentro del flujo normal
- Proporciona control fino con `continue()`, `break()`, `skipStep()`, `failStep()`
- Registra cada iteración en logs
- Maneja errores por iteración
- Anida dentro de workflows y etapas

---

## 3. Concepto Fundamental: Loop vs LoopControl

### 3.1 Dos Conceptos Distintos

**Loop** (clase):
- Contenedor que agrupa steps a repetir
- Implementa `WorkflowStep`
- Se ejecuta como un paso más del workflow
- Recibe un `LoopControl` que define las iteraciones

**LoopControl** (interfaz):
- Decide **cuándo continuar** iterando
- Se ejecuta **antes de cada iteración** para validar continuación
- Puede modificar container
- Retorna `true` (continuar) o `false` (romper loop)

### 3.2 Flujo de Ejecución

```
Loop.run()
  ├─ Adjunta LOOP_START
  ├─ Iteración 1:
  │   ├─ LoopControl.executeNextIteration(iteration=0)
  │   │   └─ Retorna true → continuar
  │   ├─ Ejecuta cada step del loop
  │   └─ Adjunta LOOP_ITERATION info
  ├─ Iteración 2:
  │   ├─ LoopControl.executeNextIteration(iteration=1)
  │   │   └─ Retorna true → continuar
  │   ├─ Ejecuta cada step del loop
  │   └─ Adjunta LOOP_ITERATION info
  ├─ LoopControl.executeNextIteration(iteration=N)
  │   └─ Retorna false → romper loop
  └─ Adjunta LOOP_END
```

---

## 4. Interfaz: LoopControl

Ubicación: [`src/Step/LoopControl.php`](../src/Step/LoopControl.php)

```php
interface LoopControl extends Describable
{
    /**
     * Return true if the next iteration of the loop shall be executed.
     * Return false to break the loop
     */
    public function executeNextIteration(
        int $iteration,
        WorkflowControl $control,
        WorkflowContainer $container
    ): bool;
}
```

### 4.1 Método: executeNextIteration

**Parámetros**:
- `int $iteration` - Número de iteración actual (0-indexed, comienza en 0)
- `WorkflowControl $control` - Para `continue()`, `break()`, `skipStep()`, `failStep()`, `warning()`
- `WorkflowContainer $container` - Compartido entre todas las iteraciones, puede modificarse

**Retorno**:
- `true` → Continuar loop (ejecutar steps de esta iteración)
- `false` → Romper loop (saltar a LOOP_END)

### 4.2 Método: getDescription (heredado de Describable)

Retorna descripción legible para logs:

```php
public function getDescription(): string
{
    return 'Process entries';
}
```

### 4.3 Responsabilidades Clave

El `LoopControl`:
1. **Valida continuación**: Decide si hay más trabajo
2. **Prepara iteración**: Puede extraer datos del container
3. **Modifica container**: Actualiza estado para el step
4. **Puede lanzar control**: `continue()`, `break()`, `skipStep()`
5. **NO ejecuta pasos**: Eso lo hace el Loop

---

## 5. Clase: Loop

Ubicación: [`src/Step/Loop.php`](../src/Step/Loop.php)

### 5.1 Constructor

```php
public function __construct(LoopControl $loopControl, bool $continueOnError = false)
{
    $this->loopControl = $loopControl;
    $this->continueOnError = $continueOnError;
}
```

**Parámetros**:
- `LoopControl $loopControl` - Injected logic que controla iteraciones
- `bool $continueOnError` - Si continuar ante errores (ver Sección 9)

### 5.2 Método: addStep

Agrega step a ejecutar en cada iteración:

```php
$loop = new Loop(myLoopControl());
$loop->addStep(new ProcessItemStep());
$loop->addStep(new ValidateProcessStep());
```

Retorna `$this` para encadenamiento.

### 5.3 Método: getDescription

Retorna descripción del LoopControl.

### 5.4 Método: run

```php
public function run(WorkflowControl $control, WorkflowContainer $container): void
{
    $iteration = 0;

    // Adjunta inicio
    $control->attachStepInfo(StepInfo::LOOP_START, 
        ['description' => $this->loopControl->getDescription()]);

    while (true) {
        try {
            // Pregunta al LoopControl si continuar
            if (!$this->loopControl->executeNextIteration($iteration, $control, $container)) {
                break;  // LoopControl decidió: NO MÁS ITERACIONES
            }

            // Ejecuta cada step del loop
            foreach ($this->steps as $step) {
                $this->wrapStepExecution($step, WorkflowState::getRunningWorkflow());
            }

            $iteration++;
        } catch (ContinueException $e) {
            // continue() en step → salta a siguiente iteración
            $iteration++;
        } catch (BreakException $e) {
            // break() en step → sale del loop
            $iteration++;
            break;
        } catch (Exception $e) {
            // Otros errores
            $iteration++;
            if (!$this->continueOnError) {
                // Propaga excepción
                throw $e;
            }
            // Si continueOnError=true: registra warning y continúa
        }

        // Adjunta info de iteración
        $control->attachStepInfo(StepInfo::LOOP_ITERATION, ['iteration' => $iteration]);
    }

    // Adjunta fin
    $control->attachStepInfo(StepInfo::LOOP_END, ['iterations' => $iteration]);
}
```

---

## 6. Patrones de Implementación de LoopControl

### 6.1 Patrón 1: Iterador Simple (Contador)

```php
class CounterLoop implements LoopControl
{
    private int $maxCount;
    private int $currentCount = 0;

    public function __construct(int $maxCount = 10)
    {
        $this->maxCount = $maxCount;
    }

    public function getDescription(): string
    {
        return "Repeat $this->maxCount times";
    }

    public function executeNextIteration(
        int $iteration,
        WorkflowControl $control,
        WorkflowContainer $container
    ): bool {
        // Retorna true si aún no hemos alcanzado el máximo
        return $iteration < $this->maxCount;
    }
}

// Uso:
$loop = new Loop(new CounterLoop(5));
$loop->addStep(new ProcessStep());
```

Ejecuta exactamente 5 iteraciones (0-4).

### 6.2 Patrón 2: Iterar sobre Array/Colección

```php
class CollectionLoop implements LoopControl
{
    private string $containerKey;
    private string $itemKey;

    public function __construct(string $containerKey, string $itemKey = 'item')
    {
        $this->containerKey = $containerKey;
        $this->itemKey = $itemKey;
    }

    public function getDescription(): string
    {
        return "Process collection: {$this->containerKey}";
    }

    public function executeNextIteration(
        int $iteration,
        WorkflowControl $control,
        WorkflowContainer $container
    ): bool {
        // Obtiene colección
        $items = $container->get($this->containerKey) ?? [];

        if (empty($items)) {
            return false;  // Sin más items → romper
        }

        // Extrae primer item
        $item = array_shift($items);
        
        // Actualiza container
        $container->set($this->itemKey, $item);
        $container->set($this->containerKey, $items);

        return true;  // Continuar para procesar este item
    }
}

// Uso:
$container = (new WorkflowContainer())
    ->set('items', ['apple', 'banana', 'cherry']);

$loop = new Loop(
    new CollectionLoop('items', 'currentItem')
);
$loop->addStep(new ProcessItemStep()); // Accede $container->get('currentItem')

$workflow = new Workflow('process-fruits')
    ->process($loop)
    ->executeWorkflow($container);
```

Ejecuta una iteración por item en la colección.

### 6.3 Patrón 3: Condición Personalizada

```php
class ConditionalLoop implements LoopControl
{
    private $condition;

    public function __construct(callable $condition)
    {
        $this->condition = $condition;
    }

    public function getDescription(): string
    {
        return "Conditional loop";
    }

    public function executeNextIteration(
        int $iteration,
        WorkflowControl $control,
        WorkflowContainer $container
    ): bool {
        // Ejecuta callable personalizado
        return ($this->condition)($iteration, $control, $container);
    }
}

// Uso:
$loop = new Loop(
    new ConditionalLoop(
        function (int $iteration, WorkflowControl $control, WorkflowContainer $container): bool {
            $maxRetries = $container->get('maxRetries') ?? 3;
            $attempts = $container->get('attempts') ?? 0;

            if ($attempts >= $maxRetries) {
                return false;  // Alcanzamos máximo
            }

            $attempts++;
            $container->set('attempts', $attempts);

            return true;  // Reintentar
        }
    )
);
$loop->addStep(new RetryableStep());
```

Permite lógica arbitraria en LoopControl.

### 6.4 Patrón 4: Reintentos con Backoff

```php
class RetryWithBackoffLoop implements LoopControl
{
    private int $maxRetries;

    public function __construct(int $maxRetries = 3)
    {
        $this->maxRetries = $maxRetries;
    }

    public function getDescription(): string
    {
        return "Retry up to $this->maxRetries times";
    }

    public function executeNextIteration(
        int $iteration,
        WorkflowControl $control,
        WorkflowContainer $container
    ): bool {
        if ($iteration > $this->maxRetries) {
            return false;  // Excedió máximo
        }

        // Backoff exponencial: 1s, 2s, 4s, 8s
        if ($iteration > 0) {
            $sleepSeconds = 2 ** ($iteration - 1);
            usleep($sleepSeconds * 1_000_000);  // Delay
        }

        return true;
    }
}
```

Implementa delays crecientes entre intentos.

---

## 7. Control de Flujo dentro de Loops

### 7.1 `continue()` - Saltar al Siguiente

```php
class ProcessWithSkipStep implements WorkflowStep
{
    public function run(WorkflowControl $control, WorkflowContainer $container): void
    {
        if ($container->get('skip')) {
            $control->continue('Condición de skip cumplida');
            return;
        }

        // Procesa normalmente
        $control->attachStepInfo('Procesando...');
    }

    public function getDescription(): string
    {
        return 'Process or skip';
    }
}
```

**Comportamiento**:
- Salta **pasos restantes de esta iteración**
- Pasa a la **siguiente iteración**
- Registra iteración como SKIPPED

### 7.2 `break()` - Salir del Loop

```php
class ProcessUntilConditionStep implements WorkflowStep
{
    public function run(WorkflowControl $control, WorkflowContainer $container): void
    {
        $count = $container->get('count') ?? 0;

        if ($count >= 10) {
            $control->break("Alcanzamos 10 items");
            return;
        }

        // Procesa
        $control->attachStepInfo("Processing item #$count");
        $container->set('count', $count + 1);
    }

    public function getDescription(): string
    {
        return 'Process until 10';
    }
}
```

**Comportamiento**:
- Sale inmediatamente del loop
- Registra iteración actual como SKIPPED
- Adjunta "Loop break in iteration #N"
- Continúa al siguiente step del workflow (fuera del loop)

### 7.3 `skipStep()` - Saltar Step Actual

```php
$loop->addStep(
    $this->setupStep(
        'validate-entry',
        function (WorkflowControl $control, WorkflowContainer $container) {
            if (!$container->get('entry')) {
                $control->skipStep('Entry is null');
                return;
            }

            // Valida...
        }
    )
);
```

**Comportamiento**:
- Step se marca como SKIPPED
- Continúa con el **siguiente step de la misma iteración**
- No afecta otras iteraciones

### 7.4 `failStep()` vs `failWorkflow()` vs General Exceptions

| Control | Comportamiento |
|---------|----------------|
| `throw Exception()` | Falla generalmente, depende de `continueOnError` |
| `failStep()` | Marca step como failed, depende de `continueOnError` |
| `failWorkflow()` | **Rompe siempre**, incluso si `continueOnError=true` |
| `skipWorkflow()` | **Rompe siempre**, marca workflow como skipped |

```php
$loop = new Loop($loopControl, $continueOnError = true);

// Caso 1: failStep() con continueOnError=true → continúa
$loop->addStep(new ErrorStep());  // failStep() → warning + continúa next item

// Caso 2: failWorkflow() → SIEMPRE rompe, incluso con continueOnError=true
$loop->addStep(new CriticalStep());  // failWorkflow() → abort loop

// Caso 3: throw Exception → depende de continueOnError
$loop->addStep(new ThrowingStep());  // throw → si continueOnError, registra warning
```

---

## 8. Flag: continueOnError

### 8.1 Comportamiento

**`continueOnError = false` (default)**:
- **Propaga excepciones** inmediatamente
- Loop interrumpe
- Workflow falla

**`continueOnError = true`**:
- **Captura excepciones de steps** (excepto control)
- Registra como warning
- Continúa loop
- Workflow **sigue exitoso** (si no hay failWorkflow)

### 8.2 Excepciones NO Afectadas por continueOnError

Las siguientes **SIEMPRE rompen el loop**:
- `FailWorkflowException` (failWorkflow)
- `SkipWorkflowException` (skipWorkflow)
- `ContinueException` (continue) - normal, sale iteración actual
- `BreakException` (break) - normal, sale del loop

### 8.3 Ejemplo: Diferencia de Comportamiento

```php
$container = (new WorkflowContainer())
    ->set('entries', ['a', null, 'c']);  // null causará error

// Opción 1: continueOnError = false (default)
$result1 = (new Workflow('test'))
    ->process(
        (new Loop($loopControl, false))
            ->addStep(new StepWithValidation())
    )
    ->executeWorkflow($container, false);

// Resultado: Fail en iteración 2
// Log: "Process: Start Loop, a, Loop iteration #1, null (failed)"
```

```php
// Opción 2: continueOnError = true
$result2 = (new Workflow('test'))
    ->process(
        (new Loop($loopControl, true))
            ->addStep(new StepWithValidation())
    )
    ->executeWorkflow($container, false);

// Resultado: Success (con warning)
// Log: "Process: Start Loop, a, Loop iteration #1, 
//              null (failed + warning), Loop iteration #2, c, Loop iteration #3"
// Warning: "Loop iteration #2 failed. Continued execution."
```

---

## 9. Logging de Loops

### 9.1 StepInfo Adjuntada

Tres tipos de StepInfo se adjuntan:

| Constante | Cuándo | Parámetros |
|-----------|--------|-----------|
| `LOOP_START` | Al inicio | `['description' => 'Descripción del loop']` |
| `LOOP_ITERATION` | Fin de cada iteración | `['iteration' => $iterationNumber]` |
| `LOOP_END` | Al completar loop | `['iterations' => $totalCount]` |

### 9.2 Log Esperado

```
Process log for workflow 'test':
Process:
  - Start Loop: ok
    - item-processor: ok
      - Procesando item a
    - Loop iteration #1: ok
    - item-processor: ok
      - Procesando item b
    - Loop iteration #2: ok
    - loop-description: ok
      - Loop finished after 2 iterations
```

### 9.3 Acceso a Logs

```php
$result = $workflow->executeWorkflow($container);
$logs = $result->debug();

// Búsqueda manual:
$lastStep = $result->getLastStep();  // Loop instance si se ejecutó
```

---

## 10. continue() en LoopControl vs en Step

### 10.1 En LoopControl: Salta Iteración Actual

```php
class LoopControlWithSkip implements LoopControl
{
    public function executeNextIteration(
        int $iteration,
        WorkflowControl $control,
        WorkflowContainer $container
    ): bool {
        if ($iteration === 1) {
            $control->continue("Skip iteración 2");
            // NO RETORNA FALSE - sigue devolviendo true/false
        }

        return $iteration < 3;  // Retorna la lógica normal
    }
}
```

**Resultado**:
- Iteración 1: Se ejecutan steps
- Iteración 2: Se salta (continue en LoopControl) → next
- Iteración 3: Se ejecutan steps
- Output: Iteración #2 marked as SKIPPED

### 10.2 En Step: Salta Pasos Restantes en Iteración

```php
class ProcessingStep implements WorkflowStep
{
    public function run(WorkflowControl $control, WorkflowContainer $container): void
    {
        if ($container->get('skip')) {
            $control->continue("Saltando pasos restantes");
        }
    }
}

// Loop tiene 3 steps
$loop->addStep(new CheckStep());
$loop->addStep(new ProcessingStep());  // ← continúe aquí
$loop->addStep(new FinalizeStep());    // ← esto se salta
```

**Resultado**:
- CheckStep: ejecuta
- ProcessingStep: continua
- FinalizeStep: **NO se ejecuta** en esta iteración
- Pasa a iteración siguiente

---

## 11. Loops Anidados

### 11.1 Loop dentro de Loop

```php
$loopOuter = new Loop(new CounterLoop(3));  // 3 iteraciones

$loopInner = new Loop(new CounterLoop(2));  // 2 iteraciones por outer

$loopInner->addStep(new ProcessInnerStep());

$loopOuter->addStep($loopInner);

$workflow = (new Workflow('nested-loops'))
    ->process($loopOuter)
    ->executeWorkflow();

// Resultado: 3 × 2 = 6 ejecuciones de ProcessInnerStep
```

### 11.2 Comportamiento de continue/break en Nesting

```
break en loop interno → sale del loop interno, continúa loop externo
continue en loop interno → siguiente iteración del loop interno
continue/break en step del loop interno → afecta solo loop interno
```

### 11.3 Container Compartido entre Loops

```php
$container = (new WorkflowContainer())
    ->set('counter', 0);

$loopOuter = new Loop(new CounterLoop(3));
$loopInner = new Loop(new CounterLoop(2));

$loopInner->addStep(
    new class implements WorkflowStep {
        public function run(WorkflowControl $control, WorkflowContainer $container): void
        {
            $count = $container->get('counter') ?? 0;
            $count++;
            $container->set('counter', $count);
            
            $control->attachStepInfo("Counter: $count");
        }

        public function getDescription(): string { return 'Increment'; }
    }
);

$loopOuter->addStep($loopInner);

(new Workflow('test'))->process($loopOuter)->executeWorkflow($container);

// Counter termina en 6 (3 iteraciones × 2 iteraciones internas)
```

---

## 12. Loop dentro de NestedWorkflow

### 12.1 Comportamiento

```php
$innerLoop = new Loop($loopControl);
$innerLoop->addStep(new ProcessStep());

$nestedWorkflow = new NestedWorkflow(
    (new Workflow('inner'))->process($innerLoop)
);

$outerWorkflow = (new Workflow('outer'))
    ->process($nestedWorkflow)
    ->executeWorkflow($container);
```

**Resultado**:
- Loop se ejecuta dentro del nested workflow
- Error en loop inner **aborta el nested workflow**
- Pero outer workflow **decide si continuar** basado en NestedWorkflow handling

### 12.2 Error Propagation

```
Outer Workflow
  → NestedWorkflow
      → Loop
          → Step (falla)
              → ¿continueOnError=true?
                  → Sí: loop continúa
                  → No: Loop propaga, NestedWorkflow falla, Outer recibe error
```

---

## 13. Loop dentro de Etapas (Before, After, etc.)

### 13.1 Loop en Etapa Validate

```php
$workflow = (new Workflow('order-validation'))
    ->validate(
        (new Loop($lineItemLoopControl))
            ->addStep(new ValidateLineItem())
    )
    ->process(new ProcessOrder());
```

**Comportamiento**:
- Loop se ejecuta en etapa Validate
- Si falla loop o step dentro → workflow no llega a Process
- Si éxito → continúa normalmente

### 13.2 Loop en Etapa After

```php
$workflow = (new Workflow('process-and-cleanup'))
    ->process(new ProcessOrder())
    ->after(
        (new Loop($cleanupLoopControl))
            ->addStep(new CleanupResource())
    );
```

**Comportamiento**:
- Loop se ejecuta al final, SIEMPRE (After ejecuta siempre)
- Errores en loop no afectan workflow (After es lo último)

---

## 14. Middleware en Loops

Los middlewares se aplican a **cada step de cada iteración**:

```php
$workflow = new Workflow('loop-with-middleware', new ProfileStep());

$loop = new Loop($loopControl);
$loop->addStep(new Step1());
$loop->addStep(new Step2());

$workflow->process($loop);

// Resultado:
// Iteration 1:
//   - ProfileStep para Step1
//   - ProfileStep para Step2
// Iteration 2:
//   - ProfileStep para Step1
//   - ProfileStep para Step2
// ... n iteraciones
```

**Performance**: Middleware × steps × iteraciones = invocaciones totales

---

## 15. Tabla de Referencia: Excepciones en Loops

| Excepción | Lanzada por | Comportamiento | Continuar con Error |
|-----------|------------|----------------|-------------------|
| `ContinueException` | `$control->continue()` | Salta a siguiente iteración | N/A (normal) |
| `BreakException` | `$control->break()` | Sale del loop | N/A (normal) |
| `SkipStepException` | `$control->skipStep()` | Salta step actual | N/A (normal) |
| `FailStepException` | `$control->failStep()` | Step falla | true: warning+continúa |
| `FailWorkflowException` | `$control->failWorkflow()` | Loop aborta | false: propaga SIEMPRE |
| `SkipWorkflowException` | `$control->skipWorkflow()` | Loop aborta | false: propaga SIEMPRE |
| General Exception | `throw ...` | Loop aborta | true: warning+continúa |

---

## 16. Antipatrones

### ❌ Antipatrón 1: Mutar Array mientras se Itera

```php
// MALO - ✗
class UnsafeLoop implements LoopControl {
    private array $items;

    public function executeNextIteration(
        int $iteration,
        WorkflowControl $control,
        WorkflowContainer $container
    ): bool {
        // Nunca modifiques el array en el LoopControl si aún lo estás iterando
        // Esto causa comportamiento impredecible
        if ($iteration > 5) {
            $this->items = [];  // ¡PELIGRO!
        }
        return !empty($this->items);
    }
}
```

### ❌ Antipatrón 2: continueOnError sin Considerar Efectos

```php
// CUIDADO - ⚠️
$loop = new Loop($loopControl, true);  // continueOnError=true
$loop->addStep(new DatabaseInsertStep());  // Si falla iteración 5 de 10
// Resultado: 9 inserts, 1 miss, inconsistencia

// MEJOR:
$loop->addStep(new DatabaseTransactionStep());  // Envuelve en transacción
```

### ❌ Antipatrón 3: Loop sin Límite Claro

```php
// MALO - ✗ (infinite loop)
class DangerousLoop implements LoopControl {
    public function executeNextIteration(
        int $iteration,
        WorkflowControl $control,
        WorkflowContainer $container
    ): bool {
        return true;  // ¡NUNCA retorna false!
    }
}

// MEJOR:
public function executeNextIteration(...): bool {
    $items = $container->get('items') ?? [];
    return !empty($items);  // Claro cuándo termina
}
```

---

## 17. Casos de Uso RealWorld

### 17.1 Procesar Fila por Fila de CSV

```php
class CSVLooperStep implements WorkflowStep
{
    private string $filePath;

    public function __construct(string $filePath)
    {
        $this->filePath = $filePath;
    }

    public function run(WorkflowControl $control, WorkflowContainer $container): void
    {
        $file = fopen($this->filePath, 'r');
        $rows = [];

        while (($row = fgetcsv($file)) !== false) {
            $rows[] = $row;
        }

        fclose($file);

        $container->set('csv_rows', $rows);
        $control->attachStepInfo("Loaded " . count($rows) . " rows");
    }

    public function getDescription(): string
    {
        return 'Load CSV file';
    }
}

class CSVRowLoop implements LoopControl
{
    public function getDescription(): string
    {
        return 'Process CSV rows';
    }

    public function executeNextIteration(
        int $iteration,
        WorkflowControl $control,
        WorkflowContainer $container
    ): bool {
        $rows = $container->get('csv_rows') ?? [];

        if (empty($rows)) {
            return false;
        }

        $row = array_shift($rows);
        $container->set('csv_rows', $rows);
        $container->set('current_row', $row);

        return true;
    }
}

// Uso:
$workflow = (new Workflow('csv-processor'))
    ->prepare(new CSVLooperStep('/path/to/data.csv'))
    ->process(
        (new Loop(new CSVRowLoop()))
            ->addStep(new ValidateRow())
            ->addStep(new ImportRow())
    )
    ->executeWorkflow();
```

### 17.2 Reintentos con Backoff

```php
class ExternalAPICallStep implements WorkflowStep
{
    private string $apiUrl;

    public function run(WorkflowControl $control, WorkflowContainer $container): void
    {
        try {
            $response = file_get_contents($this->apiUrl);
            $container->set('api_response', $response);
            $control->attachStepInfo("API call succeeded");
        } catch (Exception $e) {
            throw new Exception("API call failed: " . $e->getMessage());
        }
    }

    public function getDescription(): string
    {
        return 'Call external API';
    }
}

// Loop que reintenta con backoff
$workflow = (new Workflow('api-with-retry'))
    ->process(
        (new Loop(new RetryWithBackoffLoop(maxRetries: 3)), continueOnError: true))
            ->addStep(new ExternalAPICallStep())
    )
    ->executeWorkflow();

// Comportamiento:
// Intento 1: Immediately
// Intento 2: After 1 second
// Intento 3: After 2 seconds
// Intento 4: After 4 seconds
```

---

## 18. Hallazgos Clave

### 18.1 Loop es WorkflowStep

Loop no es etapa especial, es un WorkflowStep normal:
- Se ejecuta como cualquier step
- Puede estar en cualquier etapa
- Se afecta por middleware

### 18.2 LoopControl Determina TODO

El LoopControl es quien decide:
- Cuándo continuar
- Cuándo romper
- Cantidad de iteraciones
- Transformación de datos por iteración

### 18.3 Iteración No es Índice Directo

`$iteration` comienza en 0, NO 1:
- Iteración 0: First iteration
- Iteración 1: Second iteration
- Log muestra "Loop iteration #1" (ajustado para humanos)

### 18.4 Container es Compartido

El container **se comparte** entre iteraciones:
- Modificaciones persisten
- Ideal para contadores, accumuladores
- LoopControl debe actualizar variables que usa

### 18.5 continueOnError NO Afecta Workflow Control

- `failWorkflow()` **SIEMPRE** rompe
- `skipWorkflow()` **SIEMPRE** rompe
- Solo `failStep()` y general exceptions son afectadas

---

## 19. Referencia: Comparativa con Alternativas

| Característica | Loop | Foreach en Step | While Manual |
|----------------|------|-----------------|-------------|
| Logging integrado | ✅ | ❌ | ❌ |
| Control (continue/break) | ✅ | ❌ | ⚠️ |
| Middleware aplicado | ✅ | No (1 step) | No (1 step) |
| Error handling | ✅ (continueOnError) | Manual | Manual |
| State tracking | ✅ (container) | ✅ | ✅ |
| Readability | ✅ | ✅ | ⚠️ |

**Conclusión**: Loop es la forma idiomática de iterar en workflows.

---

## 20. Conclusiones

El sistema de loops en php-workflow es **flexible y explícito**:

### 20.1 Fortalezas

- **Loop-aware**: Framework entiende iteraciones
- **Controlable**: continue, break a nivel framework
- **Debuggable**: Logs muestran cada iteración
- **Composable**: Loops dentro de loops, nested workflows, etapas
- **Versátil**: Soporta contadores, colecciones, condiciones personalizadas

### 20.2 Responsabilidades del Usuario

- Implementar LoopControl según lógica
- Decidir `continueOnError` según tolerancia a fallos
- Manejar estado compartido en container correctamente
- Evitar infinite loops

### 20.3 Patrones Emergentes

- **Collection processing**: Típicamente con CollectionLoop
- **Retry logic**: RetryWithBackoffLoop
- **conditional**: ConditionalLoop con callable
- **generator**: Datos generados dinámicamente por iteración

### 20.4 Integración Perfecta

- Loop comienza sin sintaxis especial
- Controles (continue/break) usan API existente
- Logging automático con constantes LOOP_*
- Middleware se aplica transparentemente

---

## 21. Referencias de Código

### Ubicaciones Clave

- Clase: `src/Step/Loop.php`
- Interfaz: `src/Step/LoopControl.php`
- Tests: `tests/LoopTest.php`

### Tests Recomendados

- `testLoop()` - Basic loop iteration
- `testContinue()` - Continue en step
- `testBreak()` - Break en step
- `testMultipleStepsInLoop()` - Múltiples steps por iteración
- `testNestedWorkflowInLoop()` - NestedWorkflow dentro de loop
- `testFailInLoopWithContinueOnError()` - Error handling

---

## 22. Próximas Extensiones

Para mayor funcionalidad:

- **Grupo 6: Dependencias** - Validar precondiciones en loops
- **Grupo 8: Best Practices** - Patrones avanzados de loops
- **Grupo 10: Casos de Uso** - Ejemplos real-world completos

