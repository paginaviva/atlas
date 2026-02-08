# 14. Debugging y Troubleshooting de Workflows

## 1. Objetivo

Guía práctica para diagnosticar, depurar y resolver problemas comunes en workflows, incluyendo interpretación de logs, herramientas de debugging y patrones de troubleshooting.

---

## 2. Contexto

Los workflows pueden fallar o comportarse inesperadamente por razones que no son obvias:
- Un step se salta cuando no debería
- El workflow falla pero el error es vago
- Data desaparece entre steps
- Loop infinito o no itera como se espera

Este documento te enseña a investigar sistemáticamente.

---

## 3. Leyendo Logs: Interpretación

### 3.1 Estructura de Logs

Los logs de ejecución típicamente muestran:

```
[Workflow: checkout-process] STARTED
  [Prepare] LoadUserStep: ... 
  [Validate] EmailValidator: ✓
  [Validate] PasswordValidator: ✓
  [Before] AuthorizeStep: ... 
  [Process] ChargePaymentStep: ...
    ↳ WARNING: Retry attempt 2
    ↳ WARNING: High latency (2.3s)
  [Process] CreateOrderStep: ...
  [OnSuccess] SendEmailStep: ...
  [After] CleanupStep: ...
[Workflow: checkout-process] SUCCESS (4.2s)
```

### 3.2 Indicadores en Logs

#### ✅ Ejecución Normal

```
[Process] ProcessOrderStep: Executed successfully
  └─ INFO: Order created with ID 12345
  └─ INFO: Email scheduled for delivery
```

**Significa**: Step ejecutó completamente sin problemas

#### ⚠️ Warnings (No stops workflow)

```
[Process] SendEmailStep: Executed successfully
  └─ WARNING: Email server latency high (5s)
  └─ WARNING: Retry attempt 3/5
```

**Significa**: Step completó, pero con algún problema secundario

#### 🔴 Skip (Step ignored)

```
[Process] SendPremiumEmailStep: SKIPPED
  └─ REASON: User not premium
```

**Significa**: `$control->skipStep()` fue llamado

**Implicación**: Workflow continúa, siguiente step ejecuta

#### 🛑 Fail Step (Step fallo)

```
[Process] ProcessPaymentStep: FAILED
  └─ REASON: Insufficient funds
  └─ GOTO: OnError stage
```

**Significa**: `$control->failStep()` fue llamado o exception no manejada

**Implicación**: Workflow falla, va a OnError

#### 💥 Fail Workflow (Abort)

```
[Process] CriticalCheckStep: ABORT WORKFLOW
  └─ REASON: Security violation detected
  └─ GOTO: After (skip OnError)
```

**Significa**: `$control->failWorkflow()` fue llamado

**Implicación**: Workflow termina completamente

#### ℹ️ Info/Debug Messages

```
[Process] DataTransformStep: Executed successfully
  └─ INFO: Transformed 500 items
  └─ DEBUG: Memory used: 12.4 MB
```

**Significa**: `$control->attachStepInfo()` fue llamado

---

## 4. Errores Comunes: Mensajes y Significados

### 4.1 "WorkflowValidationException: Validation failed"

**Estructura del error**:
```
WorkflowValidationException: Validation failed
  - [email] Invalid email format
  - [phone] Email or phone required
```

**Qué significa**:
- Uno o más validators fallaron
- Nunca entró a Before/Process

**Cómo debuggear**:
```php
try {
    $result = $workflow->executeWorkflow($container);
} catch (WorkflowValidationException $e) {
    foreach ($e->getValidationErrors() as $error) {
        echo "{$error->field}: {$error->message}";
    }
}
```

**Soluciones comunes**:
1. ¿Validaste datos antes de ejecutar?
   ```php
   $container->set('email', filter_var($email, FILTER_VALIDATE_EMAIL));
   ```

2. ¿El validator espera una estructura diferente?
   ```php
   // Si validator espera object
   $container->set('user', (object)['email' => $email]);
   ```

3. ¿El validator es demasiado estricto?
   ```php
   // Considera soft validator
   $workflow->validate($strictValidator, false);  // true = hard
   ```

### 4.2 "Work Flow failed at Process stage"

**Estructura**:
```
WorkflowException: Workflow execution failed
  Step: ChargePaymentStep
  Stage: Process
  Reason: Payment gateway timeout
```

**Qué significa**:
- Un step en Process lanzó excepción
- OnError handlers ejecutaron
- Workflow marcado como fallido

**Cómo debuggear**:
```php
$result = $workflow->executeWorkflow($container, false);
$exc = $result->getException();
$step = $result->getLastStep();  // Qué step falló
echo "Failed at: " . $step->getDescription();
```

**Soluciones comunes**:

1. **Exception interna no capturada**:
   ```php
   // HAZ: Capta exception
   try {
       $result = $paymentApi->charge($amount);
   } catch (PaymentException $e) {
       $control->failStep("Payment failed: " . $e->getMessage());
   }
   
   // NO: Dejar que suba
   $result = $paymentApi->charge($amount);  // ← Exception sale
   ```

2. **Dependencia externa falló**:
   ```php
   // HAZ: Retry con backoff
   $maxRetries = 3;
   for ($i = 0; $i < $maxRetries; $i++) {
       try {
           return $externalApi->call();
       } catch (TimeoutException $e) {
           if ($i < $maxRetries - 1) {
               usleep(100000 * pow(2, $i));  // exponential backoff
               $control->warning("Retry {$i+1}/{$maxRetries}");
           } else {
               $control->failStep("Max retries exceeded");
           }
       }
   }
   ```

### 4.3 "Step skipped unexpectedly"

**Log**:
```
[Process] SendEmailStep: SKIPPED
  └─ REASON: User not premium
```

**Pero tú esperabas que ejecute**.

**Posibles causas**:
1. Condición skip cumplida
2. Dependency no satisfach
3. Loop dentro que skipea

**Cómo debuggear**:
```php
// Opción 1: Logs detallados
echo $result->getWarnings();

// Opción 2: Inspeccioná container antes
$container->set('_debug_before_email', $container->keys());

// Opción 3: Agregá step INFO antes del skip
$control->attachStepInfo('About to skip', [
    'reason' => 'is_premium = ' . $container->get('is_premium')
]);
```

**Soluciones**:
1. Verifica la condición realmente se cumple:
   ```php
   if (!$container->get('is_premium', false)) {
       $control->skipStep('Not premium');
   }
   ```

2. Usa step anterior para debug:
   ```php
   $workflow->process(new DebugStep(function($control, $container) {
       $control->attachStepInfo('Premium status', [
           'is_premium' => $container->get('is_premium')
       ]);
   }));
   ```

### 4.4 "Data container is empty"

**Síntoma**:
```php
$result = $workflow->executeWorkflow($container);
$data = $result->getContainer()->get('result');  // null or exception
```

**Qué pasó**:
- Data nunca fue guardada
- Fue deleteda después
- Key diferente de la esperada

**Cómo debuggear**:
```php
// Ver todos los keys disponibles
$keys = $result->getContainer()->keys();
print_r($keys);

// Buscar key parcial
$allData = [];
foreach ($keys as $k) {
    $allData[$k] = $result->getContainer()->get($k);
}
print_r($allData);
```

**Soluciones comunes**:
1. Step no asignó resultado:
   ```php
   // ANTES (olvida set)
   public function run(WorkflowControl $control, WorkflowContainer $container) {
       $result = $this->compute();
       // ¡Falta! $container->set('result', $result);
   }
   
   // DESPUÉS (con set)
   public function run(WorkflowControl $control, WorkflowContainer $container) {
       $result = $this->compute();
       $container->set('result', $result);  // ✓
   }
   ```

2. Key utilizó typo:
   ```php
   // Inconsistencia
   $container->set('order_id', $id);
   $value = $result->getContainer()->get('orderId');  // null
   
   // Fix: Usa mismo key
   $value = $result->getContainer()->get('order_id');  // ✓
   ```

### 4.5 "Loop runs more/fewer times than expected"

**Síntoma**:
```php
$count = 0;
$loop->while(function() use (&$count) {
    $count++;
    return $count < 5;     // Esperas 5 iteraciones
})->steps($steps);         // Pero ejecuta 3 o 10

$result = $workflow->executeWorkflow();
```

**Cómo debuggear**:
```php
$loop->while(function() use (&$count) {
    $count++;
    $control->attachStepInfo("Loop iteration", ['count' => $count]);
    return $count < 5;
});

$result = $workflow->executeWorkflow();
$info = $result->getWarnings();  // Ver ejecutadas
```

**Soluciones**:
1. **Condición incorrecta**:
   ```php
   // ❌ Terminará en 0
   $loop->while(function() {
       return 5 > 5;  // FALSE immediately
   });
   
   // ✓ Termine en 5
   $loop->while(function() {
       return $count < 5;
   });
   ```

2. **State no persiste entre iteraciones**:
   ```php
   // ❌ Count reinicia cada iteración
   $loop->while(function() {
       $count = 0;        // ← Reinicia
       return $count < 5;
   });
   
   // ✓ Count persiste en container
   $loop->while(function() {
       $count = ($container->get('loop_count') ?? 0) + 1;
       $container->set('loop_count', $count);
       return $count < 5;
   });
   ```

---

## 5. Herramientas de Debugging

### 5.1 $result->debug()

**Qué es**: Formateador completo de flujo de ejecución.

**Uso básico**:
```php
$result = $workflow->executeWorkflow($container, false);
echo $result->debug();
```

**Output**:
```
Workflow: checkout
Status: FAILED
Last Step: PaymentStep
Exception: Payment timeout

Execution Flow:
  1. Prepare: InitUser (SUCCESS)
  2. Validate: EmailValidator (SUCCESS)
  3. Before: AuthorizeUser (SUCCESS)
  4. Process: ChargePayment (FAILED) ← HERE
  5. OnError: RollbackTransaction (SUCCESS)

Warnings by Stage:
  - Process: "High latency detected"
  - Process: "Retry attempt 2"

Container State:
  - user_id: 12345
  - order_id: null (never set)
  - payment_id: null (never set)
```

**Formatos disponibles** (depend on setup):
```php
$result->debug(OutputFormat::TEXT);     // Default, texto
$result->debug(OutputFormat::JSON);     // JSON
$result->debug(OutputFormat::TABLE);    // ASCII table
```

### 5.2 ProfileStep Middleware

Framework middleware para profiling:

```php
$workflow = new Workflow('payment', new ProfileStep());

// Output:
// ProcessPaymentStep: 245ms (2.4MB memory)
// CreateOrderStep: 18ms
// SendEmailStep: 1200ms (slow!)
```

**Cómo implementar**:
```php
use Atlas\Middleware\ProfileStep;

$profile = new ProfileStep();
$workflow = new Workflow('process', $profile);

$result = $workflow->executeWorkflow();
print_r($result->getWarnings());  // Ver timings/memory
```

**En logs buscas**:
```
⏱️ ProcessPaymentStep: 245ms
💾 Memory delta: +2.4MB
⚠️ High latency warning
```

### 5.3 Custom Logger Middleware

Captura TODO lo que hace workflow:

```php
class DebugLogger implements WorkflowStep {
    public function run(WorkflowControl $control, WorkflowContainer $container) {
        echo "BEFORE: " . json_encode($container->keys());
        // Continúa...
        echo "AFTER: " . json_encode($container->keys());
    }
    
    public function getDescription(): string {
        return "Debug logging";
    }
}

$workflow = new Workflow('test', new DebugLogger());
```

### 5.4 Breakpoint Debugging

Pausa workflow en punto específico:

```php
$workflow->process(new class implements WorkflowStep {
    public function run(WorkflowControl $control, WorkflowContainer $container) {
        // Breakpoint
        if ($container->get('user_id') === 12345) {
            $control->attachStepInfo('[DEBUG] Breakpoint hit', [
                'payload' => $container->keys(),
                'backtrace' => 'check xdebug trace'
            ]);
        }
    }
    
    public function getDescription(): string { return "Breakpoint"; }
});
```

### 5.5 State Inspection Helper

Crea helper para inspeccionar estado:

```php
function inspectWorkflow(WorkflowResult $result) {
    echo "=== Workflow Debug ===\n";
    echo "Status: " . ($result->success() ? "OK" : "FAILED") . "\n";
    echo "Last Step: " . $result->getLastStep()->getDescription() . "\n";
    
    if ($result->getException()) {
        echo "Exception: " . $result->getException()->getMessage() . "\n";
    }
    
    echo "\nContainer State:\n";
    foreach ($result->getContainer()->keys() as $key) {
        $value = $result->getContainer()->get($key);
        echo "  - $key: " . json_encode($value) . "\n";
    }
    
    if ($result->hasWarnings()) {
        echo "\nWarnings:\n";
        foreach ($result->getWarnings() as $stage => $warnings) {
            foreach ($warnings as $warning) {
                echo "  - [$stage] $warning\n";
            }
        }
    }
}

$result = $workflow->executeWorkflow($container, false);
inspectWorkflow($result);
```

---

## 6. Guía de Troubleshooting: Matriz de Decisión

### 6.1 "Step no ejecutó cuando debería"

```
¿Workflow falló completamente?
├─ SÍ → Ve a "Workflow Failed"
└─ NO → ¿Step fue skipped?
    ├─ SÍ → Ver sección 4.3 (Skip)
    └─ NO → ¿Step está en rama correcta?
        ├─ SÍ → ¿Dependency no satisfecho?
        │   ├─ SÍ → Agregar dependency
        │   └─ NO → ¿Loop infinito?
        │       ├─ SÍ → Revisar condition
        │       └─ NO → Unknwon, usar debug()
        └─ NO → ¿Está en rama correcta?
            ├─ Prepare? → Se ejecuta siempre
            ├─ Validate? → Si pasa validación
            ├─ Before? → Si validó y prepare OK
            ├─ Process? → Si antes fue OK
            ├─ OnSuccess? → Si NO hubo fallo
            ├─ OnError? → Si hubo fallo
            └─ After? → Siempre
```

### 6.2 "Workflow falló pero no entiendo por qué"

**Paso 1**: Obtén resultado sin throwing
```php
$result = $workflow->executeWorkflow($container, false);
```

**Paso 2**: Chequea etapa de fallo
```php
$lastStep = $result->getLastStep();
echo $lastStep->getDescription();  // Qué step falla última?
```

**Paso 3**: Chequea exception
```php
$exc = $result->getException();
echo get_class($exc);      // Tipo de exception
echo $exc->getMessage();   // Qué dice
```

**Paso 4**: Inspecciona container
```php
foreach ($result->getContainer()->keys() as $key) {
    echo "$key = " . json_encode($result->getContainer()->get($key));
}
```

**Paso 5**: Usa debug formatter
```php
echo $result->debug();     // Full context
```

### 6.3 "Data disappears between steps"

**Checklist**:
1. ¿Un step deletea deliberadamente?
   ```php
   $container->unset('key');
   ```

2. ¿Se sobrescribe en otro step?
   ```php
   // Step A sets
   $container->set('user', $user);
   
   // Step B sobrescribe
   $container->set('user', $newUser);
   ```

3. ¿Es NestedWorkflow?
   ```php
   // Datos en nested NO auto-vuelven al parent
   $workflow->process(new NestedWorkflow($inner, $container));
   // ← Necesita extráerres explícitamente
   ```

4. ¿OnError cambió el state?
   ```php
   // OnError puede modificar container
   $workflow->onError(new RollbackStep());  // ← puede unset() data
   ```

---

## 7. Anti-patrones de Debugging

### ❌ Anti-patrón: Loguear todo a `var_export`

```php
// MAL
var_export($container->keys());  // Imprime en stdout
print_r($result->getContainer()->get('data'));
```

**Problema**: Se pierde en logs, no estructurado

**Mejor**:
```php
$control->attachStepInfo('State checkpoint', [
    'keys' => $container->keys(),
    'data' => $container->get('data')
]);

echo $result->debug();  // Formateado
```

### ❌ Anti-patrón: Exception swallowing

```php
// MAL
try {
    $result = $externalApi->call();
} catch (Exception $e) {
    // Ignorar exception
}
```

**Problema**: Nunca sabes qué falló

**Mejor**:
```php
try {
    $result = $externalApi->call();
} catch (ApiException $e) {
    $control->warning("API call failed", $e);
    $control->failStep("Could not reach external API");
}
```

### ❌ Anti-patrón: Shared state en steps

```php
// MAL
class ProcessorStep implements WorkflowStep {
    private $processed = [];  // ← Shared state!
    
    public function run(WorkflowControl $control, WorkflowContainer $container) {
        $this->processed[] = $container->get('id');
    }
}
```

**Problema**: Reutilizable? Concurrencia? Debug imposible

**Mejor**:
```php
public function run(WorkflowControl $control, WorkflowContainer $container) {
    $processed = $container->get('processed', []);
    $processed[] = $container->get('id');
    $container->set('processed', $processed);
}
```

---

## 8. Checklist de Debugging

Cuando un workflow falla y no sabes qué pasó:

- [ ] ¿Obtuviste resultado con `throwOnFailure=false`?
- [ ] ¿Llamaste `$result->getLastStep()`?
- [ ] ¿Leíste `$result->getException()->getMessage()`?
- [ ] ¿Inspeccionaste `$result->getContainer()->keys()`?
- [ ] ¿Usaste `$result->debug()` formatter?
- [ ] ¿Agregaste `attachStepInfo()` logging?
- [ ] ¿Verificaste dependencias with dependencies?
- [ ] ¿Verificaste condiciones de skip?
- [ ] ¿UsasteProfileStep` para timing?
- [ ] ¿Revisaste tests para patrón similar?

---

## 9. Debugging en Testing

### 9.1 PHPUnit Debugging

```php
public function testCheckoutFails() {
    $workflow = $this->getWorkflow();
    
    $result = $workflow->executeWorkflow($this->container, false);
    
    // Debug output en test failure
    if (!$result->success()) {
        echo "==== WORKFLOW DEBUG ====\n";
        echo $result->debug();
        echo "========================\n";
    }
    
    $this->assertFalse($result->success());
}
```

**Ejecutar con output verboso**:
```bash
phpunit --verbose tests/WorkflowTest.php
```

### 9.2 Asserts útiles

```php
$this->assertTrue($result->success(), $result->debug());
$this->assertFalse($result->success(), "Expected failure: " . $result->debug());
$this->assertTrue($result->getContainer()->has('order_id'));
$this->assertInstanceOf(PaymentException::class, $result->getException());
```

---

## 10. Resumen: Decision Tree Rápido

```
¿Hay error?
├─ No → Todo bien, nothing to debug
└─ Sí → $result = $workflow->executeWorkflow($container, false);
    ├─ $result->success() === false
    │   └─ echo $result->debug();
    │       ├─ Last Step OK?
    │       │   └─ Error en otro step → busca en trace
    │       └─ Last Step = culpable
    │           └─ Revisa step code
    │
    └─ ¿Data falta?
        └─ foreach ($result->getContainer()->keys()) { ... }
            ├─ Key SÍ existe → Chequea valor
            └─ Key NO existe
                └─ Busca dónde debía set()
                    └─ Agrega $container->set() en ese step
```

