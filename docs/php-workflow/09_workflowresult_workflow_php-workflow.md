# 9. WorkflowResult y Resultados en php-workflow

## 1. Objetivo

Documentar la API de resultados de workflows que permite acceder, inspeccionar e interpretar los resultados completos de la ejecución de un workflow, incluyendo estado de éxito, excepciones, datos finales, logs y warnings.

---

## 2. Contexto

Después de ejecutar un workflow con `executeWorkflow()`, obtienes un objeto `WorkflowResult` que es el **único punto de acceso a toda la información sobre la ejecución**:

```php
$result = $workflow->executeWorkflow($container);

// El resultado contiene TODA la información:
// - ¿Fue exitoso?
// - ¿Qué excepción ocurrió (si la hubo)?
// - ¿Cuál es el estado final del contenedor?
// - ¿Qué paso ejecutó último?
// - ¿Hubo warnings?
// - ¿Cuál es el log completo?
```

Este objeto es **central para la toma de decisiones** post-ejecución del workflow.

---

## 3. WorkflowResult - Interfaz Principal

Ubicación: [`src/State/WorkflowResult.php`](../src/State/WorkflowResult.php)

### 3.1 Constructor

```php
public function __construct(
    WorkflowState $workflowState,
    bool $success,
    ?Exception $exception = null
)
```

**Parámetros**:
- `$workflowState`: Objeto interno con estado completo
- `$success`: Boolean - ¿ejecutó sin errores?
- `$exception`: Excepción lanzada (null si exitoso)

**Nota**: No se construye manualmente, se retorna de `executeWorkflow()`.

### 3.2 Métodos Principales

#### 3.2.1 success(): bool

```php
if ($result->success()) {
    // Workflow ejecutó exitosamente
    echo "Workflow completado";
} else {
    // Workflow falló
    echo "Workflow falló";
}
```

**Retorna**:
- `true` si el workflow se ejecutó SIN excepciones en etapas críticas
- `false` si hubo excepción que llegó a `WorkflowException`

**Nota**: `true` NO significa "sin warnings". Puede haber warnings y seguir siendo exitoso.

#### 3.2.2 getException(): ?Exception

```php
if (!$result->success()) {
    $exception = $result->getException();
    
    if ($exception instanceof WorkflowException) {
        echo "Error: " . $exception->getMessage();
    }
}
```

**Retorna**:
- `Exception` que causó el fallo (o subclase específica)
- `null` si el workflow fue exitoso

**Tipos comunes**:
- `WorkflowException` - Excepción generalizada de workflow
- `WorkflowStepDependencyNotFulfilledException` - Dependencia no cumplida (PHP8+)
- Cualquier `Exception` lanzada por un step

#### 3.2.3 getContainer(): WorkflowContainer

```php
$result = $workflow->executeWorkflow($initialContainer);

// Obtiene el contenedor DESPUÉS de la ejecución
$finalContainer = $result->getContainer();

// Accede a datos modificados durante el workflow
$processedData = $finalContainer->get('result');
$userCount = $finalContainer->get('processed_count');
```

**Retorna**:
- El mismo `WorkflowContainer` pasado a `executeWorkflow()` (o uno nuevo si fue null)
- **Contiene todos los cambios** realizados durante la ejecución
- **Estado final** de todos los datos

**Importante**: Este es el contenedor compartido, modificaciones en workflow se reflejan aquí.

#### 3.2.4 getLastStep(): WorkflowStep

```php
$lastStep = $result->getLastStep();
echo "Último step ejecutado: " . $lastStep->getDescription();

// Útil para debugging - saber dónde se detuvo
if ($lastStep instanceof SpecificStep) {
    echo "Falló en SpecificStep";
}
```

**Retorna**:
- Instancia del último `WorkflowStep` que se ejecutó
- Puede ser usado para debugging (saber dónde se detuvo)
- Válido incluso si el workflow falló

**Casos**:
- Si workflow completó: último step de la etapa final (usualmente After)
- Si workflow falló: step donde ocurrió el error

#### 3.2.5 getWarnings(): array

```php
$warnings = $result->getWarnings();

foreach ($warnings as $stage => $stageWarnings) {
    echo "Stage: $stage\n";
    foreach ($stageWarnings as $warning) {
        echo "  - $warning\n";
    }
}
```

**Retorna**:
- Array asociativo: `[stageName => [warnings]]`
- Estructura: `['Prepare' => ['msg1', 'msg2'], 'Process' => ['msg3']]`
- Vacío si no hay warnings

**Casos de warning**:
- `continueOnError=true` en loop → warning al continuar
- Métodos `$control->warning()`
- Múltiples warnings posibles por etapa

#### 3.2.6 hasWarnings(): bool

```php
if ($result->hasWarnings()) {
    echo "Nota: Hubo warnings durante la ejecución";
    // Workflow exitoso pero con información adicional
}
```

**Retorna**:
- `true` si `getWarnings()` retorna array no vacío
- `false` si sin warnings

**Uso**: Verificación rápida sin iterar warnings.

#### 3.2.7 getWorkflowName(): string

```php
echo "Workflow: " . $result->getWorkflowName();  // e.g., "payment-process"
```

**Retorna**:
- Nombre que se pasó a `new Workflow($name, ...)`
- Útil para logging, debugging, identificación

#### 3.2.8 debug(?OutputFormat $formatter = null)

```php
// Con formatter default (StringLog)
echo $result->debug();

// Con formatter personalizado
$jsonFormatter = new JsonFormatter();
echo $result->debug($jsonFormatter);
```

**Retorna**:
- String formateado del log completo
- Por defecto: `StringLog` (formato legible)
- Soporta otros formatos: JSON, HTML, GraphViz, etc.

**Salida ejemplo**:
```
Process log for workflow 'payment':
Process:
  - Validate payment: ok
  - Charge card: ok
    - Charged: $99.99
OnSuccess:
  - Send receipt: ok

Summary:
  - Workflow execution: ok
    - Execution time: 234.56ms
```

---

## 4. Acceso a Datos Finales

### 4.1 Patrón: Leer Resultado

```php
$result = (new Workflow('order-process'))
    ->prepare(new LoadOrder())      // Carga $container->get('order')
    ->process(new ProcessOrder())   // Modifica $container->set('status', 'processed')
    ->executeWorkflow($container);

// Acceso a datos finales
$finalContainer = $result->getContainer();
$order = $finalContainer->get('order');      // Del prepare
$status = $finalContainer->get('status');    // Del process
```

### 4.2 Acceso a ResultadosComplejos

```php
class BatchProcessStep implements WorkflowStep {
    public function run(WorkflowControl $control, WorkflowContainer $container): void {
        // Procesa lote
        $results = ['processed' => 100, 'failed' => 5];
        $container->set('batch_results', $results);
        
        // Logs
        $control->attachStepInfo("Procesados: 100, Fallidos: 5");
    }
}

// Acceso post-ejecución
$result = $workflow->executeWorkflow();
$batchData = $result->getContainer()->get('batch_results');
echo "Procesados: " . $batchData['processed'];
```

---

## 5. Manejo de Errores y Excepciones

### 5.1 Patrón: Verificación Simple

```php
$result = $workflow->executeWorkflow($container, false);  // false = no lanzar

if ($result->success()) {
    echo "✓ Workflow completado";
    // Procesa datos
} else {
    echo "✗ Workflow falló";
    // Manejo de error
}
```

### 5.2 Patrón: Try-Catch (Lanza excepción)

```php
try {
    $result = $workflow->executeWorkflow($container);  // true = lanzar por defecto
    
    // Si llega aquí, siempre success()
    $data = $result->getContainer()->get('result');
} catch (WorkflowException $e) {
    // Workflow falló - la excepción tiene el resultado
    $result = $e->getWorkflowResult();
    echo "Falló en: " . get_class($result->getLastStep());
} catch (Exception $e) {
    // Otra excepción
}
```

### 5.3 Patrón: Inspeccionar Excepción

```php
if (!$result->success()) {
    $exception = $result->getException();
    
    if ($exception instanceof WorkflowStepDependencyNotFulfilledException) {
        echo "Error de dependencia: " . $exception->getMessage();
    } elseif ($exception instanceof FailWorkflowException) {
        echo "Workflow cancellado explícitamente";
    } else {
        echo "Error general: " . $exception->getMessage();
    }
}
```

---

## 6. WorkflowException - Excepción Wrapping

Ubicación: [`src/Exception/WorkflowException.php`](../src/Exception/WorkflowException.php)

### 6.1 Estructura

```php
class WorkflowException extends Exception
{
    private WorkflowResult $workflowResult;

    public function __construct(
        WorkflowResult $workflowResult,
        string $message,
        Exception $previous
    )
}
```

### 6.2 Cuándo se Lanza

Se lanza cuando `executeWorkflow()` es llamado con parámetro `$throwException = true` (default):

```php
// Lanza excepción si hay error
$result = $workflow->executeWorkflow($container);

// NO lanza excepción si hay error
$result = $workflow->executeWorkflow($container, false);
```

### 6.3 Recuperar WorkflowResult desde WorkflowException

```php
try {
    $result = $workflow->executeWorkflow($container);
} catch (WorkflowException $e) {
    // ¡IMPORTANTE! La excepción tiene el resultado
    $result = $e->getWorkflowResult();
    
    // Ahora puedes acceder al resultado
    $lastStep = $result->getLastStep();
    $exception = $result->getException();
    $logs = $result->debug();
}
```

### 6.4 Estructura de Nesting

```
WorkflowException (lanzada como resultado)
  ├─ $message: "Workflow 'payment' failed"
  ├─ $previous: Exception original (e.g., RuntimeException, etc.)
  └─ $workflowResult: WorkflowResult con estado completo
      ├─ success(): false
      ├─ getException(): El original que causó fallo
      ├─ getContainer(): Estado final
      └─ getLastStep(): Dónde se detuvo
```

---

## 7. Tabla de Referencia: Métodos de WorkflowResult

| Método | Retorna | Caso de Uso |
|--------|---------|------------|
| `success()` | bool | ¿Fue exitoso? |
| `getException()` | ?Exception | ¿Qué error ocurrió? |
| `getContainer()` | WorkflowContainer | ¿Cuál es el estado final? |
| `getLastStep()` | WorkflowStep | ¿Dónde se detuvo? |
| `getWarnings()` | array | ¿Qué warnings? |
| `hasWarnings()` | bool | ¿Hay warnings? |
| `getWorkflowName()` | string | ¿Cuál es el nombre? |
| `debug(?)` | string | ¿Log formateado? |

---

## 8. Estados Posibles: Matriz de Decisión

### 8.1 Estados de Éxito

```php
// ✅ EXITOSO SIN WARNINGS
$result->success() === true
$result->getException() === null
$result->hasWarnings() === false

// ✅ EXITOSO CON WARNINGS
$result->success() === true
$result->getException() === null
$result->hasWarnings() === true
// Típico en: continueOnError=true con fallos de step que se recuperan
```

### 8.2 Estado de Fallo

```php
// ❌ FALLÓ
$result->success() === false
$result->getException() !== null
$result->hasWarnings() === true/false  // Depende
// Llegó a excepción critical (failWorkflow, FailStepException no capturada, etc.)
```

### 8.3 Tabla de Estados

| Exitoso | Exception | Warnings | Significado |
|---------|-----------|----------|------------|
| true | null | false | Completado limpiamente ✅ |
| true | null | true | Completado con info ⚠️ |
| false | not null | false | Falló sin warnings ❌ |
| false | not null | true | Falló con detalles ❌ |

---

## 9. Casos de Uso Prácticos

### 9.1 E-commerce: Checkout Resultado

```php
$result = (new Workflow('checkout'))
    ->prepare(new LoadCart())
    ->validate(new ValidateCart())
    ->process(new ProcessPayment())
    ->onSuccess(new SendConfirmation())
    ->executeWorkflow($container, false);  // No lanzar

if ($result->success()) {
    // Checkout completado
    $finalContainer = $result->getContainer();
    $confirmationId = $finalContainer->get('confirmation_id');
    
    http_response_code(200);
    json_encode(['status' => 'success', 'confirmation' => $confirmationId]);
} else {
    // Checkout falló
    $exception = $result->getException();
    $failedStep = get_class($result->getLastStep());
    
    http_response_code(400);
    json_encode(['status' => 'error', 'reason' => $exception->getMessage()]);
}
```

### 9.2 Batch Processing: Inspeccionar Resultados

```php
$result = (new Workflow('batch-import'))
    ->process(new ImportRecords())
    ->onSuccess(new GenerateReport())
    ->executeWorkflow($container, false);

$finalContainer = $result->getContainer();
$imported = $finalContainer->get('imported_count') ?? 0;
$failed = $finalContainer->get('failed_count') ?? 0;

if ($result->success()) {
    echo "✓ Imported: $imported records";
    
    if ($result->hasWarnings()) {
        echo "⚠️ Warnings:\n";
        foreach ($result->getWarnings() as $stage => $warnings) {
            foreach ($warnings as $warning) {
                echo "  - $warning\n";
            }
        }
    }
} else {
    echo "✗ Import failed at: " . $result->getLastStep()->getDescription();
}
```

### 9.3 Logging Post-Ejecución

```php
$result = $workflow->executeWorkflow($container, false);

// Log completo para auditoría
$logger->info("Workflow {$result->getWorkflowName()} completed", [
    'success' => $result->success(),
    'last_step' => get_class($result->getLastStep()),
    'warnings' => count($result->hasWarnings() ? $result->getWarnings() : []),
    'exception' => $result->getException() ? get_class($result->getException()) : null,
    'debug_log' => $result->debug(),
]);
```

---

## 10. Debugging con WorkflowResult

### 10.1 Inspeccionar Estado Final

```php
$result = $workflow->executeWorkflow($container, false);

if (!$result->success()) {
    echo "=== DEBUGGING ===\n";
    
    // Dónde se detuvo
    echo "Last step: " . $result->getLastStep()->getDescription() . "\n";
    
    // Qué error
    echo "Exception: " . $result->getException()->getMessage() . "\n";
    
    // Qué datos quedan
    echo "Container keys: " . implode(", ", $result->getContainer()->keys()) . "\n";
    
    // Logs detallados
    echo "Debug output:\n" . $result->debug() . "\n";
}
```

### 10.2 Rastrear Warnings

```php
if ($result->hasWarnings()) {
    $allWarnings = $result->getWarnings();
    
    foreach ($allWarnings as $stageName => $warnings) {
        echo "Warnings in $stageName:\n";
        foreach ($warnings as $warning) {
            echo "  ⚠️ $warning\n";
        }
    }
}
```

### 10.3 Exportar para Análisis

```php
// Exportar JSON para análisis posterior
$dataForAnalysis = [
    'workflow_name' => $result->getWorkflowName(),
    'success' => $result->success(),
    'exception' => $result->getException() ? [
        'type' => get_class($result->getException()),
        'message' => $result->getException()->getMessage(),
    ] : null,
    'last_step' => get_class($result->getLastStep()),
    'warnings_count' => count($result->hasWarnings() ? $result->getWarnings() : []),
    'container_data' => $result->getContainer()->dump(),  // Custom method if exists
];

file_put_contents("workflow_result_{$timestamp}.json", json_encode($dataForAnalysis, JSON_PRETTY_PRINT));
```

---

## 11. Patrón: Wrapper para API HTTP

```php
class WorkflowApiResponse
{
    public static function fromResult(WorkflowResult $result, int $httpSuccessCode = 200)
    {
        return response()->json([
            'success' => $result->success(),
            'workflow' => $result->getWorkflowName(),
            'errors' => $result->getException() ? [
                'type' => get_class($result->getException()),
                'message' => $result->getException()->getMessage(),
            ] : null,
            'warnings' => $result->getWarnings(),
            'data' => $result->getContainer()->dump(),
            'debug' => config('app.debug') ? $result->debug() : null,
        ], $result->success() ? $httpSuccessCode : 400);
    }
}

// Uso
$result = $workflow->executeWorkflow($container, false);
return WorkflowApiResponse::fromResult($result, 201);
```

---

## 12. Precauciones y Pitfalls

### ❌ Antipatrón 1: Asumir No-Null sin Verificar

```php
// MALO - ✗
$exception = $result->getException();
$errorCode = $exception->getCode();  // ¿Puede ser null!

// BIEN - ✅
$exception = $result->getException();
if ($exception) {
    $errorCode = $exception->getCode();
}
```

### ❌ Antipatrón 2: No Revisar Warnings en Caso Exitoso

```php
// MALO - ✗ (asume éxito sin warnings)
if ($result->success()) {
    procesData($result->getContainer());
}

// BIEN - ✅ (revisa warnings incluso en éxito)
if ($result->success()) {
    if ($result->hasWarnings()) {
        logWarnings($result->getWarnings());
    }
    procesData($result->getContainer());
}
```

### ❌ Antipatrón 3: Usar getLastStep() sin Verificación

```php
// MALO - ✗
$step = $result->getLastStep();
if ($step instanceof SpecificStep) {
    // ...
}

// BIEN - ✅
$step = $result->getLastStep();
if ($step && $step instanceof SpecificStep) {
    // ...
}
```

---

## 13. Comparación: Lanzar vs No Lanzar Excepción

### 13.1 Modo No-Lanzar (recommended para APIs)

```php
$result = $workflow->executeWorkflow($container, false);

if ($result->success()) {
    // Handle success
    return ['status' => 'ok'];
} else {
    // Handle error
    return ['status' => 'error', 'error' => $result->getException()->getMessage()];
}
```

**Ventajas**:
- Tipo de retorno consistente (siempre WorkflowResult)
- Fácil para APIs HTTP
- Control claro del flujo

**Desventajas**:
- Más verboso (requiere if/else)

### 13.2 Modo Lanzar (try-catch)

```php
try {
    $result = $workflow->executeWorkflow($container);
    // Aquí sempre $result->success() === true
    return ['status' => 'ok'];
} catch (WorkflowException $e) {
    $result = $e->getWorkflowResult();
    return ['status' => 'error', 'error' => $e->getMessage()];
}
```

**Ventajas**:
- Manejo de excepciones estándar
- Fácil para usar en casos simples

**Desventajas**:
- Flujo menos explícito
- Recuperación de resultado requiere getWorkflowResult()

---

## 14. Hallazgos Clave

### 14.1 WorkflowResult es Inmutable

Una vez creado, el resultado no cambia. Refleja el estado en el momento que se creó.

### 14.2 Container es Mutable (Post-Ejecución)

Aunque el workflow terminó, puedes seguir modificando el container:

```php
$result = $workflow->executeWorkflow($container);
$finalContainer = $result->getContainer();
$finalContainer->set('new_key', 'new_value');  // ✅ Funciona
```

### 14.3 WorkflowException Contiene Todo

Si lanzas excepción, el objeto WorkflowException es tu "acceso de emergencia" al resultado:

```cpp
try {
    $result = $workflow->executeWorkflow();
} catch (WorkflowException $e) {
    $result = $e->getWorkflowResult();  // Recupera resultado completo
}
```

### 14.4 success() no Significa Sin Problemas

`success() === true` significa "no llegó a fallo crítico", NO "todo perfecto". Puede haber warnings.

---

## 15. Conclusiones

WorkflowResult es **el punto de contacto** con la ejecución:

### 15.1 Responsabilidades

- **Reportar estado**: success() o failure
- **Proporcionar detalles**: exception, last step, warnings
- **Acceso a datos**: container final
- **Debugging**: logs formatizados

### 15.2 Best Practices

1. **Siempre verificar** `success()` o usar try-catch
2. **Revisar warnings** incluso en caso exitoso
3. **Usar getContainer()** para acceder datos procesados
4. **Loguear getException()** para debugging
5. **Exportar debug()** para análisis post-mortem

### 15.3 Patrones Recomendados

- **APIs HTTP**: No-lanzar excepción, retornar status basado en `success()`
- **Scripts CLI**: Lanzar excepción para manejo de error estándar
- **Logging**: Siempre registrar `getException()` y `getWarnings()`

---

## 16. Referencias de Código

### Ubicaciones Clave

- Clase: [src/State/WorkflowResult.php](../src/State/WorkflowResult.php)
- Excepción: [src/Exception/WorkflowException.php](../src/Exception/WorkflowException.php)

### Tests Recomendados

- [tests/WorkflowTest.php](../tests/WorkflowTest.php) - Uso general
  - `testSuccessfulProcess()` - Workflow exitoso
  - `testFailingStep()` - Workflow con error
  - `testWarnings()` - Warnings en resultado

---

## 17. Próximas Extensiones

Para mayor funcionalidad:

- **Grupo 8: Best Practices** - Patrones avanzados de resultado
- **Grupo 12: Debugging** - Interpretación de resultados
- **Grupo 10: Integración** - Usar resultados en APIs

