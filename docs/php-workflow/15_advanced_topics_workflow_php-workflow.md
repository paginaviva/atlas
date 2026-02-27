# 15. Tópicos Avanzados de php-workflow

## 1. Objetivo

Explorar patrones avanzados y casos de uso sofisticados: máquinas de estado, async/concurrency, nesting profundo, y optimización de performance.

---

## 2. Contexto

Una vez que dominaste workflows básicos, necesitas:
- **Máquinas de estado** con transiciones complejas
- **Async patterns** para operaciones no-bloqueantes
- **Nesting profundo** sin perder control
- **Performance optimization** en workflow masivo

Este documento te muestra cómo.

---

## 3. State Machines: Workflows como FSM

### 3.1 Concepto

Implementar una máquina de estados usando workflows:

```
State: PENDING
  ├─ input: submit
  └─► State: PROCESSING
       ├─ input: success
       └─► State: COMPLETED
       └─ input: error
       └─► State: FAILED
```

### 3.2 Implementación Manual

```php
class OrderStateMachine {
    private Workflow $workflow;
    private WorkflowContainer $state;
    
    public function __construct() {
        $this->state = new WorkflowContainer();
        $this->state->set('status', 'PENDING');
        
        $this->workflow = new Workflow('order-fsm')
            ->process(new StateValidationStep($this->state))
            ->process(new StateTransitionStep($this->state))
            ->process(new StateActionStep($this->state))
            ->onError(new StateRollback($this->state));
    }
    
    public function transition(string $input): bool {
        $this->state->set('input', $input);
        
        $result = $this->workflow->executeWorkflow($this->state, false);
        
        return $result->success();
    }
    
    public function getStatus(): string {
        return $this->state->get('status');
    }
}

// Uso
$fsm = new OrderStateMachine();
$fsm->transition('submit');      // PENDING → PROCESSING
$fsm->transition('success');     // PROCESSING → COMPLETED
echo $fsm->getStatus();          // COMPLETED
```

### 3.3 State Transition Matrix

Estructura data de transiciones válidas:

```php
$transitionMatrix = [
    'PENDING' => [
        'submit' => 'PROCESSING',
        'cancel' => 'CANCELLED',
    ],
    'PROCESSING' => [
        'success' => 'COMPLETED',
        'error' => 'FAILED',
        'retry' => 'PROCESSING',  // Self-loop
    ],
    'COMPLETED' => [
        // No transiciones
    ],
    'FAILED' => [
        'retry' => 'PROCESSING',
    ],
];

class StateTransitionStep implements WorkflowStep {
    public function run(WorkflowControl $control, WorkflowContainer $container) {
        $current = $container->get('status');
        $input = $container->get('input');
        
        if (!isset($transitionMatrix[$current][$input])) {
            $control->failStep("Invalid transition: $current + $input");
            return;
        }
        
        $next = $transitionMatrix[$current][$input];
        $container->set('status', $next);
        $control->attachStepInfo("Transitioned to $next");
    }
    
    public function getDescription(): string {
        return "Validate and execute state transition";
    }
}
```

### 3.4 Actions por State

```php
$stateActions = [
    'PENDING' => [ProcessPaymentStep::class],
    'PROCESSING' => [VerifyPaymentStep::class, CreateOrderStep::class],
    'COMPLETED' => [SendConfirmationStep::class],
    'FAILED' => [RollbackStep::class, NotifyUserStep::class],
];

class StateActionStep implements WorkflowStep {
    public function run(WorkflowControl $control, WorkflowContainer $container) {
        $status = $container->get('status');
        
        if (!isset($stateActions[$status])) {
            return;  // No actions
        }
        
        foreach ($stateActions[$status] as $actionClass) {
            $action = new $actionClass();
            $action->run($control, $container);
            
            if (!$container->get('success', true)) {
                $control->failStep("Action failed in state $status");
                break;
            }
        }
    }
    
    public function getDescription(): string {
        return "Execute actions for current state";
    }
}
```

---

## 4. Async Patterns y Concurrency

### 4.1 Concepto: Async dentro de Sync

PHP es synchronous por naturaleza, pero puedes simular async:

```php
// Serial (predeterminado)
Step1 → Step2 → Step3  // Total: 1+2+3 = 6s

// Async pattern (usando queue)
Step1 → [Enqueue: Step2, Step3] → After
// Total: 1s + 0 (enqueued) = 1s
// Step2, Step3 ejecutan en background
```

### 4.2 Queue-Based Async

```php
class QueueDispatcher {
    private array $queue = [];
    
    public function enqueue(string $jobName, array $payload): void {
        // Store en backend (Redis, DB, etc)
        $this->queue[] = ['job' => $jobName, 'payload' => $payload];
        // O envía a queue service
    }
    
    public function dispatchQueue(): void {
        // Ejecuta background (CLI, cron, queue worker)
        foreach ($this->queue as $job) {
            // Procesa asíncronamente
        }
    }
}

// En workflow
class AsyncDispatchStep implements WorkflowStep {
    private QueueDispatcher $dispatcher;
    
    public function __construct(QueueDispatcher $dispatcher) {
        $this->dispatcher = $dispatcher;
    }
    
    public function run(WorkflowControl $control, WorkflowContainer $container) {
        $dispatcher->enqueue('send-email', [
            'to' => $container->get('email'),
            'subject' => 'Order Confirmation',
        ]);
        
        $control->attachStepInfo("Email enqueued for async processing");
    }
    
    public function getDescription(): string {
        return "Dispatch async job to queue";
    }
}

// Uso
$workflow = new Workflow('checkout')
    ->process(new ChargePaymentStep())
    ->onSuccess(new AsyncDispatchStep($dispatcher))  // No bloquea
    ->after(new CommitLogStep());
```

### 4.3 Concurrency: Múltiples Workflows Paralelos

Simula paralelismo con promises/futures:

```php
class ParallelWorkflowExecutor {
    private array $workflows = [];
    
    public function addWorkflow(string $id, Workflow $workflow, WorkflowContainer $container) {
        $this->workflows[$id] = ['workflow' => $workflow, 'container' => $container];
    }
    
    public function executeParallel(): array {
        $results = [];
        
        // En producción, usa queue system realmente paralelo
        foreach ($this->workflows as $id => $task) {
            $results[$id] = $task['workflow']->executeWorkflow(
                $task['container'],
                false
            );
        }
        
        return $results;
    }
    
    public function waitAll(): void {
        // Espera que todos terminen
        // Puede usar message bus, Redis pub/sub, etc
    }
}

// Uso: Procesa órdenes en paralelo
$executor = new ParallelWorkflowExecutor();
for ($i = 0; $i < 100; $i++) {
    $executor->addWorkflow(
        "order-$i",
        $orderWorkflow,
        new WorkflowContainer()  // Una por cada orden
    );
}

$results = $executor->executeParallel();
foreach ($results as $orderId => $result) {
    echo "$orderId: " . ($result->success() ? 'OK' : 'FAILED');
}
```

---

## 5. Deeply Nested Workflows

### 5.1 Nesting Profundo vs Flat

**Flat (3 niveles, 15 steps total)**:
```
MainWorkflow
├─ Step1, Step2, ..., Step15
```

**Nested (3 niveles de profundidad)**:
```
MainWorkflow
├─ NestedWorkflow1
│  ├─ NestedWorkflow2
│  │  ├─ NestedWorkflow3
│  │  │  ├─ FinalSteps...
│  │  └─ FinalSteps...
│  └─ FinalSteps...
└─ FinalSteps...
```

### 5.2 Por Qué: Ventajas de Nesting

1. **Organización**: Lógica separada por dominio
2. **Reutilización**: Un workflow anidado en múltiples contextos
3. **Testing**: Test cada workflow independientmente
4. **Control**: OnSuccess/OnError a nivel de sub-workflow
5. **Debugging**: Logs separados por workflow

### 5.3 Implementación: 3 Niveles

```php
// Nivel 3: Workflow más profundo
$paymentWorkflow = new Workflow('payment-processing')
    ->validate(new PaymentValidator())
    ->process(new AuthorizePaymentStep())
    ->process(new CapturePaymentStep())
    ->onError(new ReversePaymentStep());

// Nivel 2: Workflow intermedio
$checkoutWorkflow = new Workflow('checkout')
    ->validate(new CartValidator())
    ->process(new ApplyDiscountsStep())
    ->process(new NestedWorkflow($paymentWorkflow, $container))
    ->process(new CreateOrderStep());

// Nivel 1: Workflow principal
$mainWorkflow = new Workflow('user-purchase')
    ->before(new AuthenticateUserStep())
    ->process(new NestedWorkflow($checkoutWorkflow, $container))
    ->onSuccess(new SendConfirmationStep())
    ->after(new LogMetricsStep());

// Ejecución
$result = $mainWorkflow->executeWorkflow($container);
```

### 5.4 Container Inheritance

NestedWorkflows heredan del container padre:

```php
// Outer workflow
$container = new WorkflowContainer();
$container->set('user_id', 123);
$container->set('session', $session);

// Nested workflow
$nested = new NestedWorkflow($innerWorkflow, $container);

// Dentro de inner, acceso transparente:
// $innerContainer->get('user_id') → 123  ✓ (del parent)
// $innerContainer->set('order_id', 456)  → Escribe en parent también ✓
```

### 5.5 Anti-patrón: Nesting Excesivo

```php
// ❌ 10 niveles de nesting - imposible de debuggear
Main
├─ L1
│  ├─ L2
│  │  ├─ L3
│  │  │  ├─ L4 ...etc

// ✓ Máximo 3-4 niveles
// Después del nivel 4, consolida
```

**Regla**: Si tienes >4 niveles, refactoriza a steps individuales.

---

## 6. Performance Optimization

### 6.1 Bottleneck Analysis

Usa ProfileStep para identificar qué step es lento:

```php
$profile = new ProfileStep();
$workflow = new Workflow('process', $profile);

$start = microtime(true);
$result = $workflow->executeWorkflow($container);
$totalTime = microtime(true) - $start;

echo "Total: {$totalTime}ms\n";

// Logs mostrarán cada step timing:
// Step1: 10ms
// Step2: 2500ms  ← CULPABLE
// Step3: 50ms
```

### 6.2 Caché Results

No ejecutes operación costosa múltiples veces:

```php
class CachedLookupStep implements WorkflowStep {
    private $cache = [];
    
    public function run(WorkflowControl $control, WorkflowContainer $container) {
        $key = $container->get('lookup_key');
        
        // Check cache
        if (isset($this->cache[$key])) {
            $container->set('result', $this->cache[$key]);
            $control->attachStepInfo("Returned from cache");
            return;
        }
        
        // Lookup (expensive)
        $result = $this->expensiveLookup($key);
        $this->cache[$key] = $result;
        $container->set('result', $result);
        $control->attachStepInfo("Lookup completed and cached");
    }
    
    private function expensiveLookup($key) { /* ... */ }
    
    public function getDescription(): string { return "Cached lookup"; }
}
```

### 6.3 Batch Processing

Procesa items en lotes, no uno por uno:

```php
// ❌ Loop 1000 items = 1000 DB queries
$loop->for($items)
    ->steps(new InsertItemStep());      // 1000 queries

// ✓ Batch processing
class BatchInsertStep implements WorkflowStep {
    public function run(WorkflowControl $control, WorkflowContainer $container) {
        $items = $container->get('items');
        
        // Batch insert (1 query, 1000 rows)
        $this->db->insertBatch($items);
        
        $control->attachStepInfo("Inserted batch", ['count' => count($items)]);
    }
    
    public function getDescription(): string { return "Batch insert items"; }
}
```

### 6.4 Early Exit

Salta steps innecesarios:

```php
class ConditionalProcessingStep implements WorkflowStep {
    public function run(WorkflowControl $control, WorkflowContainer $container) {
        $status = $container->get('status');
        
        // Si ya procesado, skip
        if ($status === 'PROCESSED') {
            $control->skipStep('Already processed');
            return;
        }
        
        // Continue normal processing
        $this->process($container);
    }
    
    public function getDescription(): string { return "Conditional processing"; }
}
```

### 6.5 Memory Management

Suelta recursos en After:

```php
$workflow->after(new class implements WorkflowStep {
    public function run(WorkflowControl $control, WorkflowContainer $container) {
        // Limpia referencias grandes
        gc_collect_cycles();
        $control->attachStepInfo("Memory cleanup", [
            'memory_usage' => memory_get_usage(true),
        ]);
    }
    
    public function getDescription(): string { return "Memory cleanup"; }
});
```

---

## 7. Advanced Error Handling

### 7.1 Circuit Breaker Pattern

```php
class CircuitBreakerStep implements WorkflowStep {
    private int $failureThreshold = 5;
    private int $failureCount = 0;
    private bool $isOpen = false;
    
    public function run(WorkflowControl $control, WorkflowContainer $container) {
        if ($this->isOpen) {
            $control->failStep("Circuit breaker is OPEN");
            return;
        }
        
        try {
            $result = $this->callExternalService();
            $this->failureCount = 0;  // Reset
        } catch (Exception $e) {
            $this->failureCount++;
            
            if ($this->failureCount >= $this->failureThreshold) {
                $this->isOpen = true;
                $control->failWorkflow("Circuit breaker OPENED");
            } else {
                $control->warning("Failure {$this->failureCount}/{$this->failureThreshold}", $e);
            }
        }
    }
    
    public function getDescription(): string { return "Circuit breaker call"; }
    
    private function callExternalService() { /* ... */ }
}
```

### 7.2 Timeout Handling

```php
class TimeoutStep implements WorkflowStep {
    private int $timeoutMs = 5000;
    
    public function run(WorkflowControl $control, WorkflowContainer $container) {
        $startTime = microtime(true) * 1000;
        
        try {
            while (true) {
                if (microtime(true) * 1000 - $startTime > $this->timeoutMs) {
                    $control->failStep("Operation timeout ({$this->timeoutMs}ms exceeded)");
                    break;
                }
                
                // Long operation, check timeout periodically
                usleep(100000);  // 100ms check interval
            }
        } catch (Exception $e) {
            $control->failStep($e->getMessage());
        }
    }
    
    public function getDescription(): string { return "Timeout-protected operation"; }
}
```

### 7.3 Retry with Exponential Backoff

```php
class RetryWithBackoffStep implements WorkflowStep {
    private int $maxRetries = 3;
    private float $baseDelayMs = 100;
    
    public function run(WorkflowControl $control, WorkflowContainer $container) {
        $attempt = 0;
        
        while ($attempt < $this->maxRetries) {
            try {
                $result = $this->attemptOperation();
                return;  // Success
            } catch (Exception $e) {
                $attempt++;
                
                if ($attempt >= $this->maxRetries) {
                    $control->failStep($e->getMessage());
                    return;
                }
                
                $delayMs = $this->baseDelayMs * pow(2, $attempt - 1);
                usleep($delayMs * 1000);
                $control->warning("Retry attempt $attempt/$this->maxRetries");
            }
        }
    }
    
    public function getDescription(): string { return "Retry with backoff"; }
    
    private function attemptOperation() { /* ... */ }
}
```

---

## 8. Security in Advanced Workflows

### 8.1 Sensitive Data Sanitization

```php
class SanitizeStep implements WorkflowStep {
    public function run(WorkflowControl $control, WorkflowContainer $container) {
        // Remove sensitive fields before logging
        $keys = $container->keys();
        
        foreach ($keys as $key) {
            if ($this->isSensitive($key)) {
                $container->set($key, '[REDACTED]');
            }
        }
    }
    
    private function isSensitive(string $key): bool {
        return preg_match('/password|token|secret|credit_card|ssn/', $key);
    }
    
    public function getDescription(): string { return "Sanitize sensitive data"; }
}
```

### 8.2 Role-Based Access Control

```php
class RBACStep implements WorkflowStep {
    private array $allowedRoles = ['admin', 'processor'];
    
    public function run(WorkflowControl $control, WorkflowContainer $container) {
        $user = $container->get('user');
        
        if (!in_array($user->role, $this->allowedRoles)) {
            $control->failWorkflow("Insufficient permissions for user role: {$user->role}");
            return;
        }
    }
    
    public function getDescription(): string { return "Check role-based access"; }
}
```

---

## 9. Resumen: Advanced Patterns at a Glance

| Patrón | Caso de Uso | Estructura |
|--------|-----------|-----------|
| State Machine | Order states, workflow transitions | Matriz + Actions |
| Async Queue | Long operations, email, reports | Dispatcher + Queue |
| Deep Nesting | Complex domains, reusable workflows | 3-4 max levels |
| Profiling | Performance debugging | ProfileStep |
| Circuit Breaker | Unreliable services | Threshold + Toggle |
| Retry Backoff | Transient failures | Exponential delays |
| Caching | Expensive lookups | In-memory cache |
| Early Exit | Performance | Condition + Skip |

---

## 10. Próximos Pasos

Para dominar estos tópicos:
1. **Implementa**: State machine para tu dominio
2. **Perfila**: Un workflow real con ProfileStep
3. **Optimiza**: Reduce bottleneck identificado
4. **Nidifica**: Reorganiza un workflow plano
5. **Protege**: Agrega sanitización y RBAC
