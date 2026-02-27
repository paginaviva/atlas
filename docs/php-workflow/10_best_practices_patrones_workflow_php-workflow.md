# 10. Best Practices y Patrones en php-workflow

## 1. Objetivo

Documentar patrones de diseño exitosos, antipatterns a evitar, consideraciones de performance y seguridad para utilizar php-workflow de forma robusta, eficiente y mantenible en producción.

---

## 2. Contexto

Después de aprender la API de php-workflow, es crucial entender:
- **Qué SÍ es recomendable**: Patrones que facilitan mantenimiento y escalabilidad
- **Qué NO es recomendable**: Antipatterns que causan problemas y son difíciles de debuggear
- **Optimizaciones**: Cómo maximizar performance
- **Seguridad**: Cómo proteger datos y llevar workflows con responsabilidad

---

## 3. Patrón 1: Service Layer con Workflows

### 3.1 Idea Base

Workflows no contienen lógica de negocio directamente. **La lógica va en Services, los workflows orquestan**.

### 3.2 Estructura Correcta

```php
// ❌ ANTIPATRÓN - Lógica en step
class PaymentStep implements WorkflowStep {
    public function run(WorkflowControl $control, WorkflowContainer $container): void {
        // ¡Mucha lógica aquí!
        $amount = $container->get('amount');
        $card = $container->get('card');
        
        // Validación
        if ($amount < 0) throw new Exception('Invalid');
        
        // Procesamiento
        $result = /* TODO: API call */;
        
        // Más lógica...
        $container->set('payment_id', $result->id);
    }
}

// ✅ PATRÓN CORRECTO - Servicio limpio
class PaymentService {
    public function processPayment(float $amount, Card $card): PaymentResult {
        if ($amount < 0) throw new InvalidArgumentException();
        
        $result = $this->gateway->charge($amount, $card);
        
        return new PaymentResult($result->id, $result->status);
    }
}

// El step es DELGADO
class PaymentStep implements WorkflowStep {
    public function __construct(private PaymentService $service) {}
    
    public function run(WorkflowControl $control, WorkflowContainer $container): void {
        $amount = $container->get('amount');
        $card = $container->get('card');
        
        // Solo delegación
        $result = $this->service->processPayment($amount, $card);
        
        $container->set('payment_result', $result);
    }
    
    public function getDescription(): string {
        return 'Process payment with gateway';
    }
}
```

### 3.3 Ventajas

- **Testeable**: Service se testea aislado del workflow
- **Reutilizable**: Service se usa desde múltiples lugares
- **Mantenible**: Lógica centralizada
- **Step simple**: Fácil de entender

---

## 4. Patrón 2: Inyección de Dependencias

### 4.1 Construcción de Workflows

```php
// ❌ ANTIPATRÓN - Global state, hard coded
class OrderWorkflow {
    public static function create(): Workflow {
        $paymentGateway = new StripeGateway();  // Hard-coded!
        $emailService = new SmtpEmailService();  // Hard-coded!
        
        return (new Workflow('order-process'))
            ->process(new PaymentStep($paymentGateway))
            ->process(new EmailStep($emailService));
    }
}

// ✅ PATRÓN CORRECTO - Inyección externa
class OrderWorkflowFactory {
    public function __construct(
        private PaymentGateway $paymentGateway,
        private EmailService $emailService
    ) {}
    
    public function create(): Workflow {
        return (new Workflow('order-process'))
            ->process(new PaymentStep($this->paymentGateway))
            ->process(new EmailStep($this->emailService));
    }
}

// Uso con contenedor DI
$factory = $container->get(OrderWorkflowFactory::class);
$workflow = $factory->create();
```

### 4.2 Middleware Inyectado

```php
// ✅ PATRÓN - Middleware por inyección
class ProfilingWorkflow {
    public function __construct(private PaymentGateway $gateway) {}
    
    public function create(): Workflow {
        return (new Workflow('payment'))
            ->process(new PaymentStep($this->gateway))
            ->executeWorkflow(
                $container,
                middleware: [
                    new ProfileStep(),  // Injected
                    new LoggingMiddleware($this->logger),  // Injected
                ]
            );
    }
}
```

---

## 5. Patrón 3: Factory para Steps Reutilizables

### 5.1 Steps Parametrizados

```php
// ✅ PATRÓN - Factory pattern
class ValidationStepFactory {
    public static function email(string $fieldName = 'email'): WorkflowStep {
        return new class($fieldName) implements WorkflowStep {
            public function __construct(private string $field) {}
            
            public function run(WorkflowControl $control, WorkflowContainer $container): void {
                $value = $container->get($this->field);
                
                if (!filter_var($value, FILTER_VALIDATE_EMAIL)) {
                    $control->failStep("Invalid email in {$this->field}");
                }
            }
            
            public function getDescription(): string {
                return "Validate {$this->field} is email";
            }
        };
    }
    
    public static function required(string $fieldName): WorkflowStep {
        return new class($fieldName) implements WorkflowStep {
            public function __construct(private string $field) {}
            
            public function run(WorkflowControl $control, WorkflowContainer $container): void {
                if (!$container->has($this->field) || !$container->get($this->field)) {
                    $control->failStep("Required field missing: {$this->field}");
                }
            }
            
            public function getDescription(): string {
                return "Validate {$this->field} is required";
            }
        };
    }
}

// Uso
$workflow = (new Workflow('user-registration'))
    ->validate(ValidationStepFactory::required('email'))
    ->validate(ValidationStepFactory::email('email'))
    ->validate(ValidationStepFactory::required('password'));
```

---

## 6. Patrón 4: Action vs Decision Steps

### 6.1 Separación de Concerns

```php
// ✅ PATRÓN - Step de DECISIÓN (solo lee, decide)
class CheckCreditLimitStep implements WorkflowStep {
    public function run(WorkflowControl $control, WorkflowContainer $container): void {
        $user = $container->get('user');
        $amount = $container->get('amount');
        
        if ($amount > $user->creditLimit) {
            $control->failStep("Exceeds credit limit");
        }
        // No modifica nada
    }
    
    public function getDescription(): string {
        return 'Check if amount exceeds credit limit';
    }
}

// ✅ PATRÓN - Step de ACCIÓN (modifica estado)
class ApplyDiscountStep implements WorkflowStep {
    public function run(WorkflowControl $control, WorkflowContainer $container): void {
        // Lee
        $user = $container->get('user');
        $total = $container->get('total');
        
        // Calcula
        $discount = $this->calculateDiscount($user);
        
        // MODIFICA
        $container->set('discount', $discount);
        $container->set('final_total', $total - $discount);
        
        $control->attachStepInfo("Discount applied: $$discount");
    }
    
    private function calculateDiscount(User $user): float {
        return $user->isPremium() ? $user->totalSpent * 0.1 : 0;
    }
    
    public function getDescription(): string {
        return 'Apply user discount to total';
    }
}

// Patrón: DECISIÓN primero, luego ACCIÓN
$workflow = (new Workflow('checkout'))
    ->before(new CheckCreditLimitStep())      // Solo verifica
    ->process(new ApplyDiscountStep())        // Modifica
    ->process(new ChargeCardStep());          // Más modificaciones
```

---

## 7. Patrón 5: Repository Pattern Integrado

### 7.1 Flujo de Datos

```php
// ✅ PATRÓN - Repository en step
interface UserRepository {
    public function findById(string $id): ?User;
    public function save(User $user): void;
}

class LoadUserStep implements WorkflowStep {
    public function __construct(private UserRepository $repo) {}
    
    public function run(WorkflowControl $control, WorkflowContainer $container): void {
        $userId = $container->get('user_id');
        
        $user = $this->repo->findById($userId);
        if (!$user) {
            $control->failStep("User not found: $userId");
            return;
        }
        
        $container->set('user', $user);  // Acceso único al repo
    }
    
    public function getDescription(): string {
        return 'Load user from database';
    }
}

class SaveUserStep implements WorkflowStep {
    public function __construct(private UserRepository $repo) {}
    
    public function run(WorkflowControl $control, WorkflowContainer $container): void {
        $user = $container->get('user');
        
        try {
            $this->repo->save($user);
            $control->attachStepInfo("User saved: {$user->id}");
        } catch (Exception $e) {
            $control->failStep("Failed to save: " . $e->getMessage());
        }
    }
    
    public function getDescription(): string {
        return 'Save user to database';
    }
}
```

---

## 8. Antipatrón 1: Steps Stateful

### 8.1 ¿Qué es?

Un step que mantiene estado interno que cambia entre ejecuciones.

```php
// ❌ ANTIPATRÓN - Step stateful
class BadCounterStep implements WorkflowStep {
    private int $count = 0;  // Estado mutante!
    
    public function run(WorkflowControl $control, WorkflowContainer $container): void {
        $this->count++;  // Modifica interno
        $container->set('count', $this->count);
    }
    
    public function getDescription(): string {
        return 'Count invocations';
    }
}

// Primer workflow: count = 1
$workflow->executeWorkflow();

// Segundo workflow: count = 2 (¡PROBLEMA!)
$workflow->executeWorkflow();

// Tercero: count = 3 (¡Cacheó un paso anterior!)
```

### 8.2 Solución Correcta

```php
// ✅ PATRÓN - Step stateless
class GoodCounterStep implements WorkflowStep {
    public function run(WorkflowControl $control, WorkflowContainer $container): void {
        // Mantén estado solo en container
        $current = $container->get('count', 0);
        $container->set('count', $current + 1);
    }
    
    public function getDescription(): string {
        return 'Increment counter';
    }
}

// Cada ejecución es independiente
```

### 8.3 Por Qué Importa

- **Reutilización**: Same step object, múltiples workflows
- **Testing**: No hay side effects
- **Concurrencia**: Si ejecutas workflows en paralelo, datos corruptos
- **Debugging**: Más predecible

---

## 9. Antipatrón 2: Swallowing Excepciones

### 9.1 ¿Qué es?

Capturar excepciones sin hacer nada.

```php
// ❌ ANTIPATRÓN - Silenciando errores
class BadPaymentStep implements WorkflowStep {
    public function run(WorkflowControl $control, WorkflowContainer $container): void {
        try {
            $card = $container->get('card');
            $this->gateway->charge($card, 100);
        } catch (Exception $e) {
            // ¡Silenciado! Workflow continúa como si nada
        }
    }
    
    public function getDescription(): string {
        return 'Process payment';
    }
}

// Resultado: Usuario cree que pagó, workflow exitoso, tarjeta no cobrada
```

### 9.2 Solución Correcta

```php
// ✅ PATRÓN - Manejo explícito
class GoodPaymentStep implements WorkflowStep {
    public function run(WorkflowControl $control, WorkflowContainer $container): void {
        try {
            $card = $container->get('card');
            $result = $this->gateway->charge($card, 100);
            $container->set('payment_id', $result->id);
        } catch (PaymentGatewayException $e) {
            // Opción A: Falla el paso
            $control->failStep("Payment failed: " . $e->getMessage());
            
            // O Opción B: Log + retén información
            $control->warning("Payment attempt failed", $e);
            $container->set('payment_error', $e->getMessage());
        } catch (Exception $e) {
            // Excepciones inesperadas → log + falla
            $control->failStep("Unexpected error: " . $e->getMessage());
        }
    }
    
    public function getDescription(): string {
        return 'Process payment with error handling';
    }
}
```

---

## 10. Antipatrón 3: Circular Dependencies

### 10.1 ¿Qué es?

Step A requiere que Step B lo ejecute primero, y B requiere A.

```php
// ❌ ANTIPATRÓN - Dependencia circular
class PaymentStep implements WorkflowStep {
    public function run(WorkflowControl $control, WorkflowContainer $container): void {
        // Necesita 'user' del container
        $user = $container->get('user');
        // ...
    }
}

class UserStep implements WorkflowStep {
    public function run(WorkflowControl $control, WorkflowContainer $container): void {
        // Necesita 'payment_id' del container
        $paymentId = $container->get('payment_id');
        // ...
    }
}

// Workflow intenta esto:
$workflow = (new Workflow('circular'))
    ->process(new PaymentStep())    // Necesita 'user' (aún no existe)
    ->process(new UserStep());      // Necesita 'payment_id' (aún no existe)
```

### 10.2 Solución: DAG (Directed Acyclic Graph)

```php
// ✅ PATRÓN - Orden correcto (topológico)
$workflow = (new Workflow('correct'))
    ->before(new LoadUserStep())         // Proporciona: 'user'
    ->process(new PaymentStep())         // Usa: 'user', Proporciona: 'payment_id'
    ->process(new InvoiceStep());        // Usa: 'payment_id', 'user'

// Visualmente:
// LoadUser → Payment → Invoice
//   (user)    (user)    (both)
//            (payment_id)
```

### 10.3 Dokumentación de Dependencias

```php
// ✅ PATRÓN - Documentar explicítamente
class PaymentStep implements WorkflowStep {
    /**
     * Requiere en container:
     * - 'user': User object
     * - 'cart': Cart object
     *
     * Proporciona:
     * - 'payment_id': string
     * - 'payment_status': string
     */
    public function run(WorkflowControl $control, WorkflowContainer $container): void {
        // ...
    }
}

// O en PHP 8+ con Requires attribute
class PaymentStepPHP8 implements WorkflowStep {
    #[Requires('user', User::class)]
    #[Requires('cart', Cart::class)]
    public function run(WorkflowControl $control, WorkflowContainer $container): void {
        // ...
    }
}
```

---

## 11. Antipatrón 4: Mutación de Objetos Compartidos

### 11.1 ¿Qué es?

Modificar directamente objetos en el container sin copiar.

```php
// ❌ ANTIPATRÓN - Mutación directa
class BadDiscountStep implements WorkflowStep {
    public function run(WorkflowControl $control, WorkflowContainer $container): void {
        $user = $container->get('user');  // Referencia al objeto original
        $user->balance -= 100;            // ¡Modifica directamente!
        // ¿Guardó en BD? ¿Cuándo? ¿Dónde?
    }
}

// Si el workflow falla después → usuario tiene balance modificado pero sin pago
```

### 11.2 Solución: Inmutabilidad o Copia

```php
// ✅ PATRÓN A - Objetos Value Object (inmutables)
class User {
    public function __construct(
        public readonly string $id,
        public readonly float $balance,
    ) {}
    
    // No hay setters
}

class DiscountStep implements WorkflowStep {
    public function run(WorkflowControl $control, WorkflowContainer $container): void {
        $user = $container->get('user');
        
        // Crea nuevo objeto
        $updatedUser = new User(
            $user->id,
            $user->balance - 100,
        );
        
        $container->set('user', $updatedUser);  // Actualiza container
    }
}

// ✅ PATRÓN B - Objetos mutables con copia
class Order {
    public function __construct(
        public string $id,
        public float $total,
    ) {}
}

class UpdateOrderStep implements WorkflowStep {
    public function run(WorkflowControl $control, WorkflowContainer $container): void {
        $order = $container->get('order');
        $orderCopy = clone $order;  // Copia
        
        $orderCopy->total *= 0.9;   // Modifica la copia
        
        $container->set('order', $orderCopy);  // Guarda copia
    }
}
```

---

## 12. Performance: Consideraciones

### 12.1 Overhead de Logging

El sistema de logging tiene overhead. En loops intensivos, considéralo:

```php
// ⚠️ LENTO - Muchos steps en loop
$loop = new Loop();
for ($i = 0; $i < 10000; $i++) {
    $loop->addStep(new ProcessOneItemStep());  // 10,000 logs
}

// ✅ OPTIMIZADO - Batch processing
class ProcessBatchStep implements WorkflowStep {
    public function run(WorkflowControl $control, WorkflowContainer $container): void {
        $items = $container->get('items');
        $results = [];
        
        foreach ($items as $item) {
            $results[] = $this->processItem($item);  // Lógica, sin logging extra
        }
        
        $container->set('results', $results);
        $control->attachStepInfo(count($results) . " items processed");
    }
}

$workflow = (new Workflow('batch'))
    ->process(new ProcessBatchStep());  // Un log para 10,000 items
```

### 12.2 Tamaño de Logs

Los logs en memoria pueden crecer:

```php
// ⚠️ PROBLEMA - Información innecesaria en logs
class VerboseStep implements WorkflowStep {
    public function run(WorkflowControl $control, WorkflowContainer $container): void {
        // Cada item genera contexto
        foreach ($container->get('items') as $item) {
            $control->attachStepInfo("Processing item", [
                'item' => json_encode($item),  // ← Dump completo
                'timestamp' => now(),
                'memory' => memory_get_usage(),
            ]);
        }
    }
}

// ✅ OPTIMIZADO
class OptimizedStep implements WorkflowStep {
    public function run(WorkflowControl $control, WorkflowContainer $container): void {
        $items = $container->get('items');
        $processed = 0;
        $failed = 0;
        
        foreach ($items as $item) {
            // Procesa sin loguear cada uno
            if ($this->process($item)) {
                $processed++;
            } else {
                $failed++;
            }
        }
        
        // Resumen compacto
        $control->attachStepInfo("Batch complete: $processed OK, $failed failed");
    }
}
```

### 12.3 Middleware Performance

Middleware LIFO (stack) puede tener overhead si haces muchas cosas:

```php
// ⚠️ LENTO - Middleware pesado en cada step
class SlowProfilingMiddleware {
    public function __invoke(WorkflowStep $step): WorkflowStep {
        return new class($step) implements WorkflowStep {
            public function __construct(private WorkflowStep $step) {}
            
            public function run(WorkflowControl $control, WorkflowContainer $container): void {
                // Mucho trabajo aquí
                $trace = debug_backtrace();  // Costoso
                $memory_start = memory_get_usage(true);  // IO
                
                $this->step->run($control, $container);
                
                // Más procesamiento
            }
            
            public function getDescription(): string {
                return $this->step->getDescription();
            }
        };
    }
}

// ✅ OPTIMIZADO - Solo si es necesario
class FastConditionalMiddleware {
    public function __invoke(WorkflowStep $step): WorkflowStep {
        // No envuelve si no es crítico
        if (!config('profiling.enabled')) {
            return $step;  // Return as-is
        }
        
        return new class($step) implements WorkflowStep {
            public function __construct(private WorkflowStep $step) {}
            
            public function run(WorkflowControl $control, WorkflowContainer $container): void {
                $start = microtime(true);
                $this->step->run($control, $container);
                $duration = (microtime(true) - $start) * 1000;
                
                if ($duration > 100) {  // Solo log si lento
                    $control->attachStepInfo("Took {$duration}ms");
                }
            }
            
            public function getDescription(): string {
                return $this->step->getDescription();
            }
        };
    }
}
```

---

## 13. Seguridad: Validación de Entrada

### 13.1 Validar al Inicio

```php
// ✅ PATRÓN - Validación temprana
class SecureWorkflow {
    public function create(): Workflow {
        return (new Workflow('user-import'))
            ->validate(new InputSanitizationStep())     // Valida entrada
            ->validate(new SizeCheckStep())             // Tamaño máximo
            ->validate(new FormatCheckStep())           // Estructura
            ->process(new ImportDataStep())
            ->onSuccess(new SendNotificationStep());
    }
}

class InputSanitizationStep implements WorkflowStep {
    public function run(WorkflowControl $control, WorkflowContainer $container): void {
        $data = $container->get('input_data');
        
        // Validar tipo
        if (!is_array($data) && !is_object($data)) {
            $control->failStep("Input must be array or object");
            return;
        }
        
        // Validar tamaño
        if (strlen(json_encode($data)) > 10 * 1024 * 1024) {
            $control->failStep("Input too large");
            return;
        }
        
        $container->set('validated_data', $data);
    }
    
    public function getDescription(): string {
        return 'Validate and sanitize input data';
    }
}
```

### 13.2 Aislamiento de Datos

```php
// ✅ PATRÓN - Cada workflow con sus datos
class DataIsolationWorkflow {
    public function createForUser(User $user): Workflow {
        // Cada usuario obtiene su propio container
        $userContainer = new WorkflowContainer();
        $userContainer->set('user_id', $user->id);
        $userContainer->set('user_role', $user->role);
        
        return (new Workflow('user-process'))
            ->process(new ProcessUserDataStep($this->authorizer))
            ->executeWorkflow($userContainer);
    }
}

class ProcessUserDataStep implements WorkflowStep {
    public function __construct(private AuthorizerService $authorizer) {}
    
    public function run(WorkflowControl $control, WorkflowContainer $container): void {
        $userId = $container->get('user_id');
        
        // Verifica permisos
        if (!$this->authorizer->canAccess(auth()->user(), (int)$userId)) {
            $control->failStep("Unauthorized access");
            return;
        }
        
        // Solo entonces procesa
    }
    
    public function getDescription(): string {
        return 'Process user data with auth check';
    }
}
```

### 13.3 Sanitización de Logs

```php
// ✅ PATRÓN - No logguear información sensible
class SecurePaymentStep implements WorkflowStep {
    public function run(WorkflowControl $control, WorkflowContainer $container): void {
        $card = $container->get('card');
        $amount = $container->get('amount');
        
        // ✅ Seguro
        $control->attachStepInfo("Processing payment", [
            'card_last4' => substr($card->number, -4),  // Solo últimos 4 dígitos
            'amount' => $amount,
        ]);
        
        // ❌ NUNCA esto:
        // $control->attachStepInfo("Card: {$card->number}");  // ¡Número completo!
        
        $result = $this->gateway->charge($card, $amount);
        $container->set('payment_id', $result->id);
    }
    
    public function getDescription(): string {
        return 'Process payment securely';
    }
}
```

---

## 14. Testing: Best Practices

### 14.1 Testing de Steps Aislados

```php
// ✅ PATRÓN - Unit test de step
class PaymentStepTest extends TestCase {
    use WorkflowSetupTrait;
    
    public function testSuccessfulPayment(): void {
        $mockGateway = $this->createMock(PaymentGateway::class);
        $mockGateway->expects($this->once())
            ->method('charge')
            ->willReturn(new PaymentResponse('tx-123', 'success'));
        
        $step = new PaymentStep($mockGateway);
        $container = new WorkflowContainer();
        $container->set('amount', 99.99);
        $container->set('card', new TestCard());
        
        $control = $this->setupControl();
        $step->run($control, $container);
        
        $this->assertEquals('tx-123', $container->get('payment_id'));
    }
    
    public function testFailedPayment(): void {
        $mockGateway = $this->createMock(PaymentGateway::class);
        $mockGateway->method('charge')
            ->willThrowException(new PaymentGatewayException('Declined'));
        
        $step = new PaymentStep($mockGateway);
        // ... resto del test
    }
}
```

### 14.2 Testing de Workflows Completos

```php
// ✅ PATRÓN - Integration test
class CheckoutWorkflowTest extends TestCase {
    use WorkflowTestTrait, WorkflowSetupTrait;
    
    public function testCompleteCheckout(): void {
        $container = new WorkflowContainer();
        $container->set('user', new TestUser());
        $container->set('items', [new TestProduct()]);
        
        $workflow = $this->buildCheckoutWorkflow();
        $result = $workflow->executeWorkflow($container, false);
        
        $this->assertTrue($result->success());
        $this->assertNotNull($result->getContainer()->get('order_id'));
        $this->assertFalse($result->hasWarnings());
    }
    
    public function testCheckoutWithInvalidCard(): void {
        $container = new WorkflowContainer();
        $container->set('user', new TestUser());
        $container->set('card', new InvalidTestCard());
        
        $workflow = $this->buildCheckoutWorkflow();
        $result = $workflow->executeWorkflow($container, false);
        
        $this->assertFalse($result->success());
        $this->assertInstanceOf(PaymentException::class, $result->getException());
    }
}
```

---

## 15. Error Handling: Best Practices

### 15.1 Granularidad de Excepciones

```php
// ✅ PATRÓN - Excepciones específicas
class ValidatePaymentStep implements WorkflowStep {
    public function run(WorkflowControl $control, WorkflowContainer $container): void {
        try {
            $data = $this->validatePaymentData($container->get('payment'));
            $container->set('validated_payment', $data);
        } catch (InvalidCardException $e) {
            $control->failStep("Invalid card: " . $e->getMessage());
        } catch (AmountMissingException $e) {
            $control->failStep("Amount required");
        } catch (PaymentValidationException $e) {
            $control->failStep("Payment validation failed: " . $e->getMessage());
        }
    }
    
    public function getDescription(): string {
        return 'Validate payment data';
    }
}
```

### 15.2 Recovery Strategies

```php
// ✅ PATRÓN - Recuperación elegante
class ResilientWorkflow {
    public function createWithRetry(): Workflow {
        return (new Workflow('payment'))
            ->before(new LoadCartStep())
            ->process(new ValidateCartStep())
            ->process(new PaymentWithRetryStep())  // Con reintentos
            ->onError(new RollbackCartStep())      // Recuperación
            ->onSuccess(new ConfirmOrderStep());
    }
}

class PaymentWithRetryStep implements WorkflowStep {
    private const MAX_RETRIES = 3;
    
    public function run(WorkflowControl $control, WorkflowContainer $container): void {
        $amount = $container->get('amount');
        $card = $container->get('card');
        
        $lastException = null;
        for ($attempt = 1; $attempt <= self::MAX_RETRIES; $attempt++) {
            try {
                $result = $this->gateway->charge($card, $amount);
                $container->set('payment_id', $result->id);
                $control->attachStepInfo("Payment succeeded on attempt $attempt");
                return;
            } catch (TemporaryPaymentException $e) {
                $lastException = $e;
                if ($attempt < self::MAX_RETRIES) {
                    usleep(1000 * (100 * $attempt));  // Backoff exponencial
                    continue;
                }
            }
        }
        
        $control->failStep("Payment failed after $attempt attempts: " . $lastException->getMessage());
    }
    
    public function getDescription(): string {
        return 'Process payment with retries';
    }
}
```

---

## 16. Conclusiones: Rules of Thumb

### 16.1 Haz's

✅ **DO**:
1. **Mantén steps simples**: Una responsabilidad por step
2. **Inyecta dependencias**: No hard-code ni globals
3. **Documenta contracts**: Qué espera, qué proporciona
4. **Valida temprano**: En Prepare/Validate, no en Process
5. **Loguea contexto**: Información útil, no dumps completos
6. **Testea aislado**: Steps sin workflows primero
7. **Usa OnError**: Para recuperación, no try-catch silenciado
8. **Maneja excepciones**: Explícitamente en cada case
9. **Reutiliza workflows**: Factories y patterns reconocibles
10. **Monitorea performance**: Logs + métricas si es heavy

### 16.2 No Hagas

❌ **DON'T**:
1. **No mantengas estado en steps**: Stateless es mejor
2. **No silencies excepciones**: Siempre log y decide acción
3. **No modifiques objetos directamente**: Usa container
4. **No hard-codes valores**: Inyecta configuración
5. **No crees dependencias circulares**: DAG claro
6. **No logguees datos sensibles**: Sanitiza primero
7. **No ignores tamaño de logs**: En loops, usa batch
8. **No reutilices workflows sin factory**: Evita shared state
9. **No te olvides de validar entrada**: Seguridad
10. **No asumas orden sin especificarlo**: Documentar dependencias

---

## 17. Checklist de Código para PR

Antes de mergear un workflow a producción:

```markdown
- [ ] Todos los steps son stateless
- [ ] No hay hard-coded dependencias (inyectadas todas)
- [ ] Excepciones manejadas explícitamente (no swallowed)
- [ ] Datos sensibles no aparecen en logs
- [ ] Dependencias documentadas (Requires o comentarios)
- [ ] Tests que cubren happy path y error paths
- [ ] Performance razonable (sin loops innecesarios)
- [ ] OnError step definido si es crítico
- [ ] Validación en etapa Prepare/Validate
- [ ] Workflow debería ser reutilizable (factory si es needed)
```

---

## 18. Referencias

### Ubicaciones Clave

- Tests: [tests/WorkflowTest.php](../tests/WorkflowTest.php)
- Patrón Service Layer: [src/Middleware/ProfileStep.php](../src/Middleware/ProfileStep.php)
- Manejo de errores: [src/Exception/](../src/Exception/)

### Lecturas Relacionadas

- [Document 02 - Step Execution](02_definicion_pasos_workflow_php-workflow.md)
- [Document 04 - Control Flow](04_control_flujo_avanzado_workflow_php-workflow.md)
- [Document 06 - Middleware](06_middleware_extensibilidad_workflow_php-workflow.md)

