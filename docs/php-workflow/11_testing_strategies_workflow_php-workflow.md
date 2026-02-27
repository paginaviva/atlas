# 11. Testing Strategies para Workflows en php-workflow

## 1. Objetivo

Documentar estrategias de testing para workflows, cómo testear steps de forma aislada, cómo integrar workflows completos, qué helpers proporciona php-workflow, y patrones recomendados para cobertura completa.

---

## 2. Contexto

Testear workflows correctamente requiere:
- **Unit testing**: Steps aislados sin workflow
- **Integration testing**: Workflows completos
- **Mocking**: Dependencias externas (servicios, bases de datos)
- **Assertions**: Verificar resultados, logs, estado
- **Helpers**: php-workflow proporciona utilidades que facilitan testing

---

## 3. Helpers Disponibles: WorkflowTestTrait

### 3.1 assertDebugLog()

Verifica que el output de logs coincida con lo esperado.

```php
use PHPWorkflow\Tests\WorkflowTestTrait;

class MyWorkflowTest extends TestCase {
    use WorkflowTestTrait;
    
    public function testDebugOutput(): void {
        $result = $workflow->executeWorkflow();
        
        $this->assertDebugLog(
            <<<DEBUG
            Process log for workflow 'test':
            Process:
              - my-step: ok
                - Info added
            
            Summary:
              - Workflow execution: ok
                - Execution time: *
            DEBUG,
            $result
        );
    }
}
```

**Parámetros**:
- `string $expected`: Log esperado (con `*` para wildcard en tiempos)
- `WorkflowResult $result`: Resultado del workflow
- `?OutputFormat $formatter`: Formateador personalizado (default: StringLog)

**Características**:
- Normaliza clases anónimas
- Normaliza tiempos (cualquier Ms es `*`)
- Útil para verificar que el workflow ejecutó las etapas correctas

### 3.2 expectFailAtStep()

Verifica que workflow falló exactamente en un step específico.

```php
public function testFailsAtPayment(): void {
    $result = $workflow->executeWorkflow($container, false);
    
    $this->expectFailAtStep(PaymentStep::class, $result);
}
```

**Assertions implícitas**:
- `success() === false`
- `getLastStep()` es exactamente esa clase

### 3.3 expectSkipAtStep()

Verifica que workflow fue exitoso pero skip ocurrió en un step específico.

```php
public function testSkipsValidation(): void {
    $result = $workflow->executeWorkflow($container);
    
    $this->expectSkipAtStep(ValidationStep::class, $result);
}
```

**Assertions implícitas**:
- `success() === true`
- `getLastStep()` es exactamente esa clase

---

## 4. Helpers Disponibles: WorkflowSetupTrait

### 4.1 setupEmptyStep()

Crea un step que no hace nada (para testing).

```php
use PHPWorkflow\Tests\WorkflowSetupTrait;

class MyTest extends TestCase {
    use WorkflowSetupTrait;
    
    public function test(): void {
        $step = $this->setupEmptyStep('my-description');
        
        $workflow = (new Workflow('test'))
            ->process($step)
            ->executeWorkflow();
        
        $this->assertTrue($workflow->success());
    }
}
```

**Usa**: Cuando necesitas steps que no modifiquen estado, solo avancen.

### 4.2 setupStep()

Crea un step personalizado con comportamiento definido por callable.

```php
public function testPaymentProcessing(): void {
    $paymentStep = $this->setupStep(
        'process-payment',
        function (WorkflowControl $control, WorkflowContainer $container) {
            $amount = $container->get('amount');
            
            if ($amount < 0) {
                $control->failStep('Invalid amount');
                return;
            }
            
            $container->set('payment_id', 'tx-' . uniqid());
            $control->attachStepInfo("Charged $amount");
        }
    );
    
    $container = (new WorkflowContainer())->set('amount', 99.99);
    $result = (new Workflow('checkout'))
        ->process($paymentStep)
        ->executeWorkflow($container);
    
    $this->assertTrue($result->success());
    $this->assertNotNull($result->getContainer()->get('payment_id'));
}
```

**Parámetros**:
- `string $description`: Descripción del step
- `callable $callable`: Función con firma `(WorkflowControl, WorkflowContainer): void`

**Retorna**: `WorkflowStep` listo para usar en workflows

### 4.3 setupLoop()

Crea un LoopControl personalizado para testeos.

```php
public function testLoopProcessing(): void {
    $loop = $this->setupLoop(
        'item-loop',
        function (WorkflowControl $control, WorkflowContainer $container): bool {
            $items = $container->get('items');
            
            if (empty($items)) {
                return false;  // Stop looping
            }
            
            $item = array_shift($items);
            $container->set('items', $items);
            $container->set('current_item', $item);
            
            return true;  // Continue looping
        }
    );
    
    $container = (new WorkflowContainer())->set('items', ['a', 'b', 'c']);
    $result = (new Workflow('process-items'))
        ->process((new Loop($loop))->addStep($this->setupEmptyStep('process')))
        ->executeWorkflow($container);
    
    $this->assertTrue($result->success());
}
```

### 4.4 failDataProvider()

Proporciona ways diferentes de fallar un step (para data-driven tests).

```php
/**
 * @dataProvider failDataProvider
 */
public function testHandlesFailures(callable $failingStep): void {
    $result = (new Workflow('test'))
        ->process($this->setupStep('failing', $failingStep))
        ->executeWorkflow(null, false);
    
    $this->assertFalse($result->success());
}
```

**Proporciona 3 formas de fallo**:
1. Lanzar excepción
2. Llamar `failStep()`
3. Llamar `failWorkflow()`

### 4.5 Helpers de Contenedor

```php
// Helper: loop con entrada
$loopControl = $this->entryLoopControl();  // Devuelve LoopControl predefinido

// Helper: step de procesamiento
$processStep = $this->processEntry();  // Devuelve WorkflowStep predefinido
```

---

## 5. Testing Pattern 1: Unit Test de Step Aislado

### 5.1 Test Simple

```php
class PaymentStepTest extends TestCase {
    use WorkflowSetupTrait;
    
    public function testSuccessfulPayment(): void {
        // Arrange
        $step = $this->setupStep(
            'payment',
            function (WorkflowControl $control, WorkflowContainer $container) {
                $card = $container->get('card');
                $amount = $container->get('amount');
                
                if (!$card || $amount <= 0) {
                    $control->failStep('Invalid input');
                    return;
                }
                
                $container->set('payment_id', 'tx-123');
            }
        );
        
        // Act
        $container = (new WorkflowContainer())
            ->set('card', new TestCard())
            ->set('amount', 99.99);
        
        $control = $this->createMock(WorkflowControl::class);
        $step->run($control, $container);
        
        // Assert
        $this->assertEquals('tx-123', $container->get('payment_id'));
    }
}
```

### 5.2 Test con Mock de Dependencias

```php
class ValidateUserStepTest extends TestCase {
    public function testValidatesUserExists(): void {
        // Arrange
        $mockUserRepo = $this->createMock(UserRepository::class);
        $mockUserRepo->expects($this->once())
            ->method('findById')
            ->with('user-123')
            ->willReturn(new User('user-123', 'John'));
        
        $step = new ValidateUserStep($mockUserRepo);
        
        // Act
        $container = (new WorkflowContainer())->set('user_id', 'user-123');
        $control = $this->createMock(WorkflowControl::class);
        
        $step->run($control, $container);
        
        // Assert
        $control->expects('failStep')->never();
    }
    
    public function testFailsWhenUserNotFound(): void {
        // Arrange
        $mockUserRepo = $this->createMock(UserRepository::class);
        $mockUserRepo->method('findById')->willReturn(null);
        
        $step = new ValidateUserStep($mockUserRepo);
        
        // Act
        $container = (new WorkflowContainer())->set('user_id', 'invalid');
        $control = $this->createMock(WorkflowControl::class);
        
        // Assert
        $control->expects($this->once())
            ->method('failStep')
            ->with('User not found');
        
        $step->run($control, $container);
    }
}
```

### 5.3 Testing Exceptions en Steps

```php
class HandleExceptionStepTest extends TestCase {
    public function testCatchesServiceException(): void {
        // Arrange
        $mockService = $this->createMock(PaymentService::class);
        $mockService->method('charge')
            ->willThrowException(new PaymentGatewayException('Declined'));
        
        $step = new PaymentStep($mockService);
        
        // Act
        $container = (new WorkflowContainer())->set('amount', 100);
        $control = $this->createMock(WorkflowControl::class);
        
        $control->expects($this->once())
            ->method('failStep')
            ->with($this->stringContains('Declined'));
        
        // Assert - no exception thrown
        $step->run($control, $container);
    }
}
```

---

## 6. Testing Pattern 2: Integration Test de Workflow Completo

### 6.1 Test Happy Path

```php
class CheckoutWorkflowTest extends TestCase {
    use WorkflowSetupTrait, WorkflowTestTrait;
    
    public function testCompleteCheckout(): void {
        // Arrange
        $container = (new WorkflowContainer())
            ->set('user', new TestUser('john', 'john@example.com'))
            ->set('items', [
                new TestProduct('ITEM1', 29.99),
                new TestProduct('ITEM2', 49.99),
            ])
            ->set('card', new TestCard('4111111111111111'));
        
        // Act
        $workflow = $this->buildCheckoutWorkflow();
        $result = $workflow->executeWorkflow($container);
        
        // Assert
        $this->assertTrue($result->success());
        $this->assertFalse($result->hasWarnings());
        
        $finalContainer = $result->getContainer();
        $this->assertNotNull($finalContainer->get('order_id'));
        $this->assertEquals(79.98, $finalContainer->get('total'));
    }
    
    private function buildCheckoutWorkflow(): Workflow {
        return (new Workflow('checkout'))
            ->before($this->setupStep('validate-cart', fn() => null))
            ->process($this->setupStep('calculate-total', function(WC $control, WC $c) {
                $total = array_reduce($c->get('items'), fn($sum, $item) => $sum + $item->price, 0);
                $c->set('total', $total);
            }))
            ->process($this->setupStep('process-payment', fn() => null))
            ->process($this->setupStep('create-order', function(WC $control, WC $c) {
                $c->set('order_id', 'ORD-' . uniqid());
            }))
            ->onSuccess($this->setupStep('send-confirmation', fn() => null));
    }
}
```

### 6.2 Test Error Path

```php
public function testCheckoutFailsWithoutCard(): void {
    // Arrange
    $container = (new WorkflowContainer())
        ->set('items', [new TestProduct('X', 99)])
        ->set('card', null);  // Missing card
    
    // Act
    $workflow = $this->buildCheckoutWorkflow();
    $result = $workflow->executeWorkflow($container, false);  // No throw
    
    // Assert
    $this->assertFalse($result->success());
    $this->assertNotNull($result->getException());
}
```

### 6.3 Test con Validación

```php
public function testValidationErrorsCollected(): void {
    // Arrange
    $container = (new WorkflowContainer())
        ->set('email', 'invalid-email')
        ->set('password', '123');  // Too short
    
    // Act
    $workflow = (new Workflow('register'))
        ->validate($this->setupStep('validate-email', function(WC $c, WC $cnt) {
            if (!filter_var($cnt->get('email'), FILTER_VALIDATE_EMAIL)) {
                $c->failStep('Invalid email');
            }
        }))
        ->validate($this->setupStep('validate-password', function(WC $c, WC $cnt) {
            if (strlen($cnt->get('password')) < 8) {
                $c->failStep('Password too short');
            }
        }));
    
    $result = $workflow->executeWorkflow($container, false);
    
    // Assert
    $this->assertFalse($result->success());
    $exception = $result->getException();
    $this->assertInstanceOf(WorkflowValidationException::class, $exception);
    $this->assertCount(2, $exception->getValidationErrors());
}
```

---

## 7. Testing Pattern 3: Data-Driven Tests

### 7.1 Con Data Provider

```php
class PaymentProcessingTest extends TestCase {
    use WorkflowSetupTrait;
    
    /**
     * @dataProvider paymentAmounts
     */
    public function testHandlesVariousAmounts(float $amount, bool $shouldSucceed): void {
        $result = (new Workflow('payment'))
            ->process($this->setupStep('validate-amount', function(WC $c, WC $cnt) use ($amount) {
                if ($amount <= 0 || $amount > 10000) {
                    $c->failStep('Amount out of range');
                }
            }))
            ->executeWorkflow(
                (new WorkflowContainer())->set('amount', $amount),
                false
            );
        
        $this->assertEquals($shouldSucceed, $result->success());
    }
    
    public function paymentAmounts(): array {
        return [
            'Valid small amount' => [10.00, true],
            'Valid large amount' => [9999.99, true],
            'Zero' => [0, false],
            'Negative' => [-100, false],
            'Over limit' => [10001, false],
        ];
    }
}
```

### 7.2 Con Iteration

```php
/**
 * @dataProvider userStatusProvider
 */
public function testAuthorizationByStatus(string $status, bool $canAccess): void {
    $step = $this->setupStep('check-access', function(WC $c, WC $cnt) use ($status) {
        $permissions = [
            'admin' => true,
            'moderator' => true,
            'user' => false,
        ];
        
        if (!$permissions[$status] ?? false) {
            $c->failStep('Access denied');
        }
    });
    
    $result = (new Workflow('access-check'))
        ->process($step)
        ->executeWorkflow(
            (new WorkflowContainer())->set('status', $status),
            false
        );
    
    $this->assertEquals($canAccess, $result->success());
}

public function userStatusProvider(): array {
    return [
        'admin' => ['admin', true],
        'moderator' => ['moderator', true],
        'user' => ['user', false],
        'guest' => ['guest', false],
    ];
}
```

---

## 8. Testing Pattern 4: Loops

### 8.1 Test Loop Simple

```php
class LoopCompleteTest extends TestCase {
    use WorkflowSetupTrait, WorkflowTestTrait;
    
    public function testLoopIteratesCorrectly(): void {
        // Arrange
        $container = (new WorkflowContainer())->set('items', ['a', 'b', 'c']);
        
        // Act
        $result = (new Workflow('loop-test'))
            ->process(
                (new Loop($this->setupLoop('items-loop', function(WC $c, WC $cnt) {
                    $items = $cnt->get('items');
                    if (empty($items)) return false;
                    
                    $item = array_shift($items);
                    $cnt->set('items', $items);
                    $cnt->set('current_item', $item);
                    return true;
                })))
                ->addStep($this->setupStep('process', function(WC $c, WC $cnt) {
                    $item = $cnt->get('current_item');
                    $c->attachStepInfo("Processing $item");
                }))
            )
            ->executeWorkflow($container);
        
        // Assert
        $this->assertTrue($result->success());
        $this->assertDebugLog(
            <<<DEBUG
            Process log for workflow 'loop-test':
            Process:
              - Start Loop: ok
                - process: ok
                  - Processing a
                - Loop iteration #1: ok
                - process: ok
                  - Processing b
                - Loop iteration #2: ok
                - process: ok
                  - Processing c
                - Loop iteration #3: ok
                - items-loop: ok
                  - Loop finished after 3 iterations
            
            Summary:
              - Workflow execution: ok
                - Execution time: *
            DEBUG,
            $result,
        );
    }
}
```

### 8.2 Test Loop con Break

```php
public function testLoopWithBreak(): void {
    $result = (new Workflow('loop-break'))
        ->process(
            (new Loop($this->setupLoop('counter-loop', function(WC $c, WC $cnt) {
                $count = ($cnt->get('count') ?? 0) + 1;
                $cnt->set('count', $count);
                return $count < 5;  // Loop 5 times
            })))
            ->addStep($this->setupStep('check', function(WC $c, WC $cnt) {
                if ($cnt->get('count') === 3) {
                    $c->break('Reached target');
                }
                $c->attachStepInfo("Iteration " . $cnt->get('count'));
            }))
        )
        ->executeWorkflow();
    
    $this->assertTrue($result->success());
    // Loop stopped at iteration 3 due to break
}
```

---

## 9. Testing Pattern 5: Nested Workflows

### 9.1 Test NestedWorkflow

```php
class NestedWorkflowTest extends TestCase {
    use WorkflowSetupTrait, WorkflowTestTrait;
    
    public function testNestedWorkflowInheritance(): void {
        // Arrange
        $innerWorkflow = (new Workflow('rate-user'))
            ->process($this->setupStep('rate', function(WC $c, WC $cnt) {
                $rating = $cnt->get('rating');
                if ($rating < 1 || $rating > 5) {
                    $c->failStep('Invalid rating');
                }
                $cnt->set('rating_saved', true);
            }));
        
        $outerWorkflow = (new Workflow('main'))
            ->before($this->setupStep('load-user', function(WC $c, WC $cnt) {
                $cnt->set('user_id', 'user-123');
            }))
            ->process(new NestedWorkflow($innerWorkflow))
            ->process($this->setupStep('confirm', function(WC $c, WC $cnt) {
                $this->assertTrue($cnt->get('rating_saved'));
            }));
        
        // Act
        $container = (new WorkflowContainer())->set('rating', 4);
        $result = $outerWorkflow->executeWorkflow($container);
        
        // Assert
        $this->assertTrue($result->success());
        // Inner workflow data heredado por outer
        $this->assertEquals('user-123', $result->getContainer()->get('user_id'));
    }
}
```

---

## 10. Testing Pattern 6: Middleware

### 10.1 Test Middleware Application

```php
class MiddlewareTestingTest extends TestCase {
    use WorkflowSetupTrait;
    
    public function testMiddlewareExecutes(): void {
        // Arrange - Track if middleware ran
        $middlewareRan = false;
        
        $middleware = function(WorkflowStep $step): WorkflowStep {
            return new class($step) implements WorkflowStep {
                public function __construct(private WorkflowStep $inner) {}
                
                public function run(WC $c, WC $cnt): void {
                    // Middleware logic
                    $cnt->set('middleware_ran', true);
                    $this->inner->run($c, $cnt);
                }
                
                public function getDescription(): string {
                    return $this->inner->getDescription();
                }
            };
        };
        
        // Act
        $result = (new Workflow('test'))
            ->process($this->setupEmptyStep('test-step'))
            ->executeWorkflow(null, middleware: [$middleware]);
        
        // Assert
        $this->assertTrue($result->getContainer()->get('middleware_ran'));
    }
}
```

---

## 11. Testing Anti-Patterns

### ❌ Anti-Patrón 1: Tocar WorkflowState Directamente

```php
// ❌ NO HAGAS ESTO
public function testBadAccess(): void {
    $result = $workflow->executeWorkflow();
    
    // Direct access to WorkflowState (if available)
    $state = $result->getState();  // ← No existe, es private
}
```

### ❌ Anti-Patrón 2: Tests sin Assertions

```php
// ❌ NO HAGAS ESTO
public function testWorkflow(): void {
    $workflow->executeWorkflow();
    // No assertions!
}
```

### ❌ Anti-Patrón 3: Tests Interdependientes

```php
// ❌ NO HAGAS ESTO
class PaymentTests extends TestCase {
    private $paymentId;
    
    public function testStep1Payment(): void {
        $this->paymentId = 'tx-123';
    }
    
    public function testStep2Confirmation(): void {
        // Depende de que testStep1Payment haya corrido
        $result = $workflow->executeWorkflow(
            (new WC())->set('payment_id', $this->paymentId)
        );
        // Tests deben ser independientes
    }
}
```

### ✅ Solución: Tests Independientes

```php
// ✅ BIEN
class PaymentTests extends TestCase {
    public function testPaymentProcessing(): void {
        $result = (new Workflow('payment'))
            ->process(new PaymentStep())
            ->executeWorkflow($this->createContainer('tx-123'));
        
        $this->assertTrue($result->success());
    }
    
    public function testPaymentConfirmation(): void {
        // No depende de test anterior
        $result = (new Workflow('confirm'))
            ->process(new ConfirmationStep())
            ->executeWorkflow($this->createContainer('tx-123'));
        
        $this->assertTrue($result->success());
    }
    
    private function createContainer(string $paymentId): WorkflowContainer {
        return (new WorkflowContainer())->set('payment_id', $paymentId);
    }
}
```

---

## 12. Checklist de Testing

Antes de mergear código:

```markdown
## Unit Tests
- [ ] Cada step probado aisladamente
- [ ] Happy path testedo (success)
- [ ] Error path testedo (failures)
- [ ] Edge cases cubiertos
- [ ] Mocks en lugar de dependencias reales

## Integration Tests
- [ ] Workflow completo testedo
- [ ] Stages verificadas (Prepare/Process/OnSuccess/OnError)
- [ ] Data flow correcto (container propagación)
- [ ] Logs verificados (assertDebugLog)

## Data-Driven Tests
- [ ] DataProvider usado para variaciones
- [ ] Múltiples inputs cubiertos
- [ ] Boundary conditions testadas

## Coverage Goals
- [ ] Min 80% cobertura de código
- [ ] 100% de paths críticos
- [ ] OnError paths incluidos
- [ ] Exception scenarios cubiertos

## Non-Functional
- [ ] Tests ejecutan rápido (< 1s por test)
- [ ] Sin acceso a BD/APIs reales
- [ ] Sin dependencias externas
- [ ] Determinísticos (mismo resultado siempre)
```

---

## 13. Mejores Prácticas

### 13.1 Arrange-Act-Assert

```php
public function testUserRegistration(): void {
    // ARRANGE - Setup data y mocks
    $container = (new WorkflowContainer())
        ->set('email', 'user@example.com')
        ->set('password', 'secure123');
    
    $mockUserRepo = $this->createMock(UserRepository::class);
    $mockUserRepo->expects($this->once())->method('create');
    
    $workflow = $this->buildRegistrationWorkflow($mockUserRepo);
    
    // ACT - Execute el workflow
    $result = $workflow->executeWorkflow($container);
    
    // ASSERT - Verify resultados
    $this->assertTrue($result->success());
    $this->assertNotNull($result->getContainer()->get('user_id'));
}
```

### 13.2 Factory de Workflows para Tests

```php
class WorkflowFactory {
    public static function buildCheckoutForTest(
        UserRepository $userRepo,
        PaymentGateway $gateway
    ): Workflow {
        return (new Workflow('checkout'))
            ->before(new LoadUserStep($userRepo))
            ->process(new CalculateTotalStep())
            ->process(new PaymentStep($gateway))
            ->onSuccess(new CreateOrderStep());
    }
}

// En test
public function testCheckout(): void {
    $mockUserRepo = $this->createMock(UserRepository::class);
    $mockGateway = $this->createMock(PaymentGateway::class);
    
    $workflow = WorkflowFactory::buildCheckoutForTest($mockUserRepo, $mockGateway);
    $result = $workflow->executeWorkflow();
}
```

### 13.3 Custom Assertions

```php
class WorkflowAssertions {
    public static function assertWorkflowSucceeded(WorkflowResult $result): void {
        static::assertTrue($result->success(), 
            "Expected workflow to succeed but got exception: " . 
            ($result->getException() ? $result->getException()->getMessage() : 'unknown')
        );
    }
    
    public static function assertStepWasLast(string $stepClass, WorkflowResult $result): void {
        static::assertSame($stepClass, get_class($result->getLastStep()),
            "Expected last step to be $stepClass, got " . get_class($result->getLastStep())
        );
    }
}

// Uso
class MyTest extends TestCase {
    public function test(): void {
        $result = $workflow->executeWorkflow();
        WorkflowAssertions::assertWorkflowSucceeded($result);
        WorkflowAssertions::assertStepWasLast(LastStep::class, $result);
    }
}
```

---

## 14. Performance Testing

### 14.1 Test Ejecución Rápida

```php
public function testProcessingPerformance(): void {
    $items = array_fill(0, 1000, ['id' => 1]);
    $container = (new WorkflowContainer())->set('items', $items);
    
    $start = microtime(true);
    $result = $workflow->executeWorkflow($container);
    $duration = microtime(true) - $start;
    
    $this->assertTrue($result->success());
    $this->assertLessThan(1, $duration, 'Processing should take < 1 second');
}
```

### 14.2 Memory Testing

```php
public function testMemoryUsage(): void {
    $items = array_fill(0, 100000, new LargeObject());
    
    $memBefore = memory_get_usage(true);
    $result = $workflow->executeWorkflow(
        (new WorkflowContainer())->set('items', $items)
    );
    $memAfter = memory_get_usage(true);
    
    $memDiff = ($memAfter - $memBefore) / 1024 / 1024;  // MB
    
    $this->assertLessThan(100, $memDiff, 'Memory usage should be < 100MB');
}
```

---

## 15. Referencias

### Ubicaciones Clave

- [tests/WorkflowTestTrait.php](../tests/WorkflowTestTrait.php) - Assertions disponibles
- [tests/WorkflowSetupTrait.php](../tests/WorkflowSetupTrait.php) - Helpers de setup
- [tests/WorkflowTest.php](../tests/WorkflowTest.php) - Exemplos reales
- [tests/StepDependencyTest.php](../tests/StepDependencyTest.php) - Testing con dependencias
- [tests/LoopTest.php](../tests/LoopTest.php) - Testing loops

### Tests Recomendados para Estudio

1. **testMinimalWorkflow()** - Workflow más simple
2. **testFailingSoftValidationsAreCollected()** - Error handling
3. **testLoop()** - Loop processing
4. **testMissingKeyFails()** - Dependency checking

